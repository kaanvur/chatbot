<script lang="ts">
	import { env } from '$env/dynamic/public';
	import { HfInference } from '@huggingface/inference';

	type Mesaj = {
		rol: 'kullanici' | 'ai';
		icerik: string;
	};

	const API_TOKEN = env.PUBLIC_HF_API_TOKEN;
	let mesajlar: Mesaj[] = [];
	let giris = '';
	let yukleniyor = false;
	let modelYukleniyor = false;
	let modelHataMesaji = '';

	const client = new HfInference(API_TOKEN);

	const MODELLER = [
		{
			id: 'mistralai/Mistral-Nemo-Instruct-2407',
			ad: 'Mistral-Nemo-Instruct',
			dil: 'Çok Dilli',
			tip: 'chat'
		},
		{
			id: 'mistralai/Mistral-7B-Instruct-v0.2',
			ad: 'Mistral-7B-Instruct',
			dil: 'Çok Dilli',
			tip: 'chat'
		},
		{
			id: 'ytu-ce-cosmos/turkish-gpt2',
			ad: 'YTU Turkish-GPT2',
			dil: 'Türkçe',
			tip: 'text'
		}
	];
	let secilenModel = MODELLER[0].id;

	interface HFError {
		error: { message: string };
	}

	async function mesajGonder() {
		if (!giris.trim() || yukleniyor) return;

		try {
			yukleniyor = true;
			modelYukleniyor = false;
			modelHataMesaji = '';

			const secilenModelBilgisi = MODELLER.find(m => m.id === secilenModel);
			const yeniMesaj = { rol: 'kullanici', icerik: giris } as Mesaj;
			mesajlar = [...mesajlar, yeniMesaj];
			
			const aiMesaj = { rol: 'ai', icerik: '' } as Mesaj;
			mesajlar = [...mesajlar, aiMesaj];
			
			giris = '';

			if (secilenModelBilgisi?.tip === 'chat') {
				const stream = await client.chatCompletionStream({
					model: secilenModel,
					messages: [{ role: 'user', content: yeniMesaj.icerik }],
					max_tokens: 500
				});

				let yanit = '';
				for await (const chunk of stream) {
					if (chunk.choices && chunk.choices.length > 0) {
						const yeniIcerik = chunk.choices[0].delta.content;
						yanit += yeniIcerik;
						mesajlar = mesajlar.map((m, index) => 
							index === mesajlar.length - 1 ? { ...m, icerik: yanit } : m
						);
					}
				}
			} else {
				// Text generation modelleri için
				const stream = await client.textGenerationStream({
					model: secilenModel,
					inputs: yeniMesaj.icerik,
					parameters: {
						max_new_tokens: 500,
						temperature: 0.7
					}
				});

				let yanit = '';
				for await (const chunk of stream) {
					if (chunk.token.text) {
						yanit += chunk.token.text;
						mesajlar = mesajlar.map((m, index) => 
							index === mesajlar.length - 1 ? { ...m, icerik: yanit } : m
						);
					}
				}
			}

		} catch (error) {
			console.error(error);
			modelHataMesaji = 'Bir hata oluştu. Lütfen tekrar deneyin.';
			mesajlar = mesajlar.slice(0, -1);
		} finally {
			yukleniyor = false;
		}
	}
</script>

<div class="max-w-3xl mx-auto p-4 bg-white rounded-2xl shadow-md">
	<div class="mb-4 flex gap-2 items-center">
		<select
			id="model"
			bind:value={secilenModel}
			class="flex-1 p-2 border border-gray-200 rounded-lg text-sm bg-white disabled:bg-gray-100 disabled:cursor-not-allowed"
			disabled={yukleniyor}
		>
			{#each MODELLER as model}
				<option value={model.id}>
					{model.ad} ({model.dil})
				</option>
			{/each}
		</select>
	</div>

	{#if modelHataMesaji}
		<div class="bg-red-100 border border-red-200 text-red-800 p-3 rounded-lg mb-4 text-center">
			{modelHataMesaji}
		</div>
	{/if}

	<div class="h-[500px] overflow-y-auto p-4 border border-gray-200 rounded-lg mb-4">
		{#each mesajlar as mesaj}
			<div
				class="mb-4 p-4 rounded-lg max-w-[80%] {mesaj.rol === 'kullanici'
					? 'ml-auto bg-blue-500 text-white'
					: 'mr-auto bg-gray-100 text-gray-800'}"
			>
				<div class="mesaj-icerik">
					{mesaj.icerik}
				</div>
			</div>
		{/each}

		{#if yukleniyor}
			<div class="text-center text-gray-500 p-2 italic">
				<span>Yanıt bekleniyor...</span>
			</div>
		{:else if modelHataMesaji}
			<div class="bg-red-100 border border-red-200 text-red-800 p-3 rounded-lg mb-4 text-center">
				{modelHataMesaji}
			</div>
		{/if}
	</div>

	<form on:submit|preventDefault={mesajGonder} class="flex gap-2">
		<input
			type="text"
			bind:value={giris}
			placeholder="Mesajınızı yazın..."
			disabled={yukleniyor}
			class="flex-1 p-3 border border-gray-200 rounded-lg text-base disabled:bg-gray-50"
		/>
		<button
			type="submit"
			disabled={yukleniyor || !giris.trim()}
			class="px-6 py-3 bg-blue-500 text-white font-medium rounded-lg transition-colors hover:bg-blue-600 disabled:bg-blue-300 disabled:cursor-not-allowed"
		>
			{modelYukleniyor ? 'Model Yükleniyor...' : 'Gönder'}
		</button>
	</form>
</div>
