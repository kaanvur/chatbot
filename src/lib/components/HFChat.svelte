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
			id: 'facebook/xglm-564M',
			ad: 'XGLM-564M',
			dil: 'Türkçe Destekli'
		},
		{
			id: 'ytu-ce-cosmos/Turkish-Llama-8b-DPO-v0.1',
			ad: 'Turkish-Llama-8B',
			dil: 'Türkçe'
		},
		{
			id: 'redrussianarmy/gpt2-turkish-cased',
			ad: 'GPT2-Turkish-Cased',
			dil: 'Türkçe'
		},
		{
			id: 'ytu-ce-cosmos/turkish-gpt2',
			ad: 'YTU Turkish-GPT2',
			dil: 'Türkçe'
		}
	];
	let secilenModel = MODELLER[0].id;

	async function mesajGonder() {
		if (!giris.trim() || yukleniyor) return;

		try {
			yukleniyor = true;
			modelYukleniyor = false;
			modelHataMesaji = '';

			const yeniMesaj = { rol: 'kullanici', icerik: giris } as Mesaj;
			mesajlar = [...mesajlar, yeniMesaj];

			const parameters = secilenModel.includes('opus-mt')
				? {
						max_length: 500,
						temperature: 0.7,
						top_p: 0.95
					}
				: {
						max_new_tokens: 500,
						temperature: 0.7,
						top_p: 0.95,
						return_full_text: false
					};

			const prompt = `<s>[INST] ${giris} [/INST]`;

			const yanit = await fetch(`https://api-inference.huggingface.co/models/${secilenModel}`, {
				method: 'POST',
				headers: {
					Authorization: `Bearer ${API_TOKEN}`,
					'Content-Type': 'application/json'
				},
				body: JSON.stringify({
					inputs: prompt,
					parameters
				})
			});

			const data = await yanit.json();

			if (data.error) {
				modelYukleniyor = false;
				modelHataMesaji = data.error;
				if (data.estimated_time) {
					modelYukleniyor = true;
					modelHataMesaji = `Model yükleniyor... Tahmini süre: ${Math.ceil(data.estimated_time)} saniye`;
					setTimeout(() => {
						yukleniyor = false;
						mesajGonder();
					}, data.estimated_time * 1000);
				}
				return;
			}

			if (Array.isArray(data) && data.length > 0) {
				const aiYanit = data[0].generated_text;
				mesajlar = [
					...mesajlar,
					{
						rol: 'ai',
						icerik: aiYanit
					}
				];
				giris = '';
			}
		} catch (error) {
			console.error('Hata:', error);
			modelHataMesaji = 'Bir hata oluştu. Lütfen tekrar deneyin.';
		} finally {
			if (!modelYukleniyor) {
				yukleniyor = false;
			}
		}
	}
</script>

<div class="chat-container">
	<div class="model-secici">
		<select id="model" bind:value={secilenModel} class="model-select" disabled={yukleniyor}>
			{#each MODELLER as model}
				<option value={model.id}>
					{model.ad} ({model.dil})
				</option>
			{/each}
		</select>
	</div>

	{#if modelHataMesaji}
		<div class="hata-mesaji">
			{modelHataMesaji}
		</div>
	{/if}

	<div class="mesajlar">
		{#each mesajlar as mesaj}
			<div class="mesaj {mesaj.rol}">
				<div class="mesaj-icerik">
					{mesaj.icerik}
				</div>
			</div>
		{/each}

		{#if yukleniyor}
			<div class="yukleniyor">
				<span>Yanıt bekleniyor...</span>
			</div>
		{/if}
	</div>

	<form on:submit|preventDefault={mesajGonder} class="giris-alani">
		<input type="text" bind:value={giris} placeholder="Mesajınızı yazın..." disabled={yukleniyor} />
		<button type="submit" disabled={yukleniyor || !giris.trim()}>
			{modelYukleniyor ? 'Model Yükleniyor...' : 'Gönder'}
		</button>
	</form>
</div>

<style>
	.chat-container {
		max-width: 800px;
		margin: 0 auto;
		padding: 1rem;
		background-color: #ffffff;
		border-radius: 1rem;
		box-shadow: 0 4px 6px -1px rgb(0 0 0 / 0.1);
	}

	.mesajlar {
		height: 500px;
		overflow-y: auto;
		padding: 1rem;
		border: 1px solid #e5e7eb;
		border-radius: 0.5rem;
		margin-bottom: 1rem;
	}

	.mesaj {
		margin-bottom: 1rem;
		padding: 1rem;
		border-radius: 0.5rem;
		max-width: 80%;
	}

	.kullanici {
		margin-left: auto;
		background-color: #3b82f6;
		color: white;
	}

	.ai {
		margin-right: auto;
		background-color: #f3f4f6;
		color: #1f2937;
	}

	.giris-alani {
		display: flex;
		gap: 0.5rem;
	}

	input {
		flex: 1;
		padding: 0.75rem;
		border: 1px solid #e5e7eb;
		border-radius: 0.5rem;
		font-size: 1rem;
	}

	button {
		padding: 0.75rem 1.5rem;
		background-color: #3b82f6;
		color: white;
		border: none;
		border-radius: 0.5rem;
		font-weight: 500;
		cursor: pointer;
		transition: background-color 0.2s;
	}

	button:hover {
		background-color: #2563eb;
	}

	button:disabled {
		background-color: #93c5fd;
		cursor: not-allowed;
	}

	.model-secici {
		margin-bottom: 1rem;
		display: flex;
		gap: 0.5rem;
		align-items: center;
	}

	.model-select {
		flex: 1;
		padding: 0.5rem;
		border: 1px solid #e5e7eb;
		border-radius: 0.5rem;
		font-size: 0.875rem;
		background-color: white;
	}

	.yukleniyor {
		text-align: center;
		color: #6b7280;
		padding: 0.5rem;
		font-style: italic;
	}

	.hata-mesaji {
		background-color: #fee2e2;
		border: 1px solid #fecaca;
		color: #991b1b;
		padding: 0.75rem;
		border-radius: 0.5rem;
		margin-bottom: 1rem;
		text-align: center;
	}

	.model-select:disabled {
		background-color: #f3f4f6;
		cursor: not-allowed;
	}
</style>
