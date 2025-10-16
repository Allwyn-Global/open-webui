<script lang="ts">
	import { page } from '$app/stores';
	import { goto } from '$app/navigation';
	import Chat from '$lib/components/chat/Chat.svelte';
	import ModelLanding from '$lib/components/landing/ModelLanding.svelte';

	// Show landing page if no chat ID and no model specified
	$: showLanding = !$page.params.id && !$page.url.searchParams.get('model');

	// Redirect to form pages for videogen and audiogen using reactive statement
	$: {
		const model = $page.url.searchParams.get('model');

		// Check if this is the video generator and not coming from a sessionStorage message
		if ((model?.toLowerCase().includes('video') || model === 'videogen') &&
		    !sessionStorage.getItem('videogen_initial_message')) {
			goto('/videogen');
		}
		// Check if this is the audio generator and not coming from a sessionStorage message
		else if ((model?.toLowerCase().includes('voice') || model === 'audiogen') &&
		         !sessionStorage.getItem('audiogen_initial_message')) {
			goto('/audiogen');
		}
	}
</script>

{#if showLanding}
	<ModelLanding />
{:else}
	<Chat />
{/if}
