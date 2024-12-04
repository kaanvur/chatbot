<script lang="ts">
	import { onMount } from 'svelte';
	import { env } from '$env/dynamic/public';

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

	const MODELLER = [
		{
			id: 'mistralai/Mistral-7B-Instruct-v0.2',
			ad: 'Mistral-7B-Instruct',
			dil: 'Çok Dilli'
		},
		{
			id: 'openai-community/gpt2',
			ad: 'GPT-2',
			dil: 'Çok Dilli'
		},
		{
			id: 'ytu-ce-cosmos/turkish-gpt2',
			ad: 'YTU Turkish-GPT2',
			dil: 'Türkçe'
		}
	];
	let secilenModel = MODELLER[0].id;

	async function query(modelId: string, prompt: string) {
		const parameters ={ max_new_tokens: 500, temperature: 0.7, top_p: 0.95, return_full_text: false };

		const response = await fetch(`https://api-inference.huggingface.co/models/${modelId}`, {
				method: 'POST',
				headers: {
					Authorization: `Bearer ${API_TOKEN}`,
				'Content-Type': 'application/json',
				},
			body: JSON.stringify({ inputs: prompt, parameters }),
			});

		if (!response.ok) {
			throw new Error(`HTTP error! status: ${response.status}`);
		}

		return await response.json();
	}

	async function mesajGonder() {
		if (!giris.trim() || yukleniyor) return;

		try {
			yukleniyor = true;
				modelYukleniyor = false;
			modelHataMesaji = '';

			const yeniMesaj = { rol: 'kullanici', icerik: giris } as Mesaj;
			mesajlar = [...mesajlar, yeniMesaj];

			const data = await query(secilenModel, giris);
			if (data.error) {
				modelYukleniyor = false;
				modelHataMesaji = data.error;
				return;
			}

			if (Array.isArray(data) && data.length > 0) {
				const aiYanit = data[0].generated_text;
				mesajlar = [...mesajlar, { rol: 'ai', icerik: aiYanit }];
				giris = '';
			}
		} catch (error) {
			console.error(error);
			modelHataMesaji = 'Bir hata oluştu. Lütfen tekrar deneyin.';
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
