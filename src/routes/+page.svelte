<script>
    import HFChat from '$lib/components/HFChat.svelte';
</script>

<main>
    <h1>Hugging Face Chat</h1>
    <HFChat />
</main>

<style>
    main {
        min-height: 100vh;
        padding: 2rem;
        background-color: #f9fafb;
    }

    h1 {
        text-align: center;
        margin-bottom: 2rem;
        color: #1f2937;
        font-size: 2rem;
    }
</style>