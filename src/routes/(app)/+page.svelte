<script lang="ts">
	import { page } from '$app/stores';
	import Chat from '$lib/components/chat/Chat.svelte';
	import ModelLanding from '$lib/components/landing/ModelLanding.svelte';

	// Show landing page if no chat ID and no model specified
	$: showLanding = !$page.params.id && !$page.url.searchParams.get('model');
</script>

{#if showLanding}
	<ModelLanding />
{:else}
	<Chat />
{/if}
