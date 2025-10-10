<script lang="ts">
	import { getContext, onMount } from 'svelte';
	import { goto } from '$app/navigation';
	import { models, showSidebar } from '$lib/stores';
	import { getModels, getPipelines } from '$lib/apis';
	import { getFunctions } from '$lib/apis/functions';
	import MenuLines from '../icons/MenuLines.svelte';
	import type { i18n as i18nType } from 'i18next';
	const i18n: Writable<i18nType> = getContext('i18n');
	import type { Writable } from 'svelte/store';

	let availableModels = [];
	let filteredModels = [];
	let selectedTags = [];
	let searchQuery = '';

	// Get all unique tags from dynamically loaded models
	$: allTags = [...new Set(filteredModels.map(model => getModelInfo(model.id).tags || []).flat())].sort();

	// Filter models based on selected tags and search query
	$: {
		filteredModels = availableModels.filter(model => {
			const modelInfo = getModelInfo(model.id);

			// Check tag filter (AND logic - model must have ALL selected tags)
			const hasSelectedTags = selectedTags.length === 0 ||
				selectedTags.every(tag => (modelInfo.tags || []).includes(tag));

			// Check search filter
			const matchesSearch = searchQuery === '' ||
				modelInfo.name.toLowerCase().includes(searchQuery.toLowerCase()) ||
				modelInfo.description.toLowerCase().includes(searchQuery.toLowerCase());

			return hasSelectedTags && matchesSearch;
		});
	}

	const toggleTag = (tag) => {
		if (selectedTags.includes(tag)) {
			selectedTags = selectedTags.filter(t => t !== tag);
		} else {
			selectedTags = [...selectedTags, tag];
		}
	};

	const clearAllFilters = () => {
		selectedTags = [];
		searchQuery = '';
	};

	onMount(async () => {

		try {
			// Get auth token from localStorage
			const token = localStorage.token || '';

			// Fetch models, functions, and pipelines in parallel
			const [fetchedModels, functions, pipelines] = await Promise.all([
				getModels(token).catch(() => []),
				getFunctions(token).catch(() => []),
				getPipelines(token).catch(() => [])
			]);

			// Combine all available items
			let allItems = [];

			// Add models
			if (fetchedModels && fetchedModels.length > 0) {
				allItems = [...allItems, ...fetchedModels];
			}

			// Add functions - format them to match model structure
			if (functions && functions.length > 0) {
				const functionItems = functions.map(func => ({
					id: func.id,
					name: func.name || func.id,
					type: 'function'
				}));
				allItems = [...allItems, ...functionItems];
			}

			// Add pipelines - format them to match model structure
			if (pipelines && pipelines.length > 0) {
				const pipelineItems = pipelines.map(pipeline => ({
					id: pipeline.id,
					name: pipeline.name || pipeline.id,
					type: 'pipeline'
				}));
				allItems = [...allItems, ...pipelineItems];
			}

			// Deduplicate by ID to prevent duplicates
			const uniqueItems = [];
			const seenIds = new Set();

			for (const item of allItems) {
				if (!seenIds.has(item.id)) {
					seenIds.add(item.id);
					uniqueItems.push(item);
				}
			}

			// Use only dynamically fetched items
			availableModels = uniqueItems;
		} catch (error) {
			// If all APIs fail, show empty state instead of hardcoded fallback
			availableModels = [];
		}
	});

	const startChatWithModel = (modelId: string) => {
		// Check if this is the video generator - route to form instead
		if (modelId.toLowerCase().includes('video') || modelId === 'videogen') {
			goto('/videogen');
			return;
		}

		// Navigate to chat with the selected model
		goto(`/?model=${modelId}`);
	};

	const getModelInfo = (modelId: string) => {
		const model = availableModels.find(m => m.id === modelId);

		// Handle functions
		if (model?.type === 'function') {
			// Marketing code function variants
			if (modelId.toLowerCase().includes('marketing')) {
				return {
					emoji: '📊',
					name: 'Marketing Code',
					description: 'Reviews campaign concepts and visuals against the Allwyn Marketing Code',
					specs: 'Brand compliance • Campaign review',
					tags: ['AI Chat', 'Marketing']
				};
			}

			// Audio generation function
			if (modelId.toLowerCase().includes('audio')) {
				return {
					emoji: '🎵',
					name: 'Audio Generator',
					description: 'Create high-quality audio content from text scripts or prompts',
					specs: 'Conversational • Audio Synthesis',
					tags: ['Audio', 'Creative']
				};
			}

			// User researcher function
			if (modelId.toLowerCase().includes('researcher')) {
				return {
					emoji: '🔍',
					name: 'User Researcher',
					description: 'Expert user researcher for fast, unmoderated testing via self-serve research platforms',
					specs: 'Research focused • User testing',
					tags: ['Research']
				};
			}

			// Default function
			return {
				emoji: '⚙️',
				name: model.name || modelId.replace(/[_.:]/g, ' '),
				description: 'Custom function for enhanced capabilities',
				specs: 'Function • Custom logic',
				tags: ['Workflow']
			};
		}

		// Handle pipelines
		if (model?.type === 'pipeline') {
			// N8N workflow for image generation
			if (modelId.toLowerCase().includes('n8n') || modelId.toLowerCase().includes('workflow') || modelId.toLowerCase().includes('image')) {
				return {
					emoji: '🎨',
					name: 'Image Generator',
					description: 'Generate, edit and animate your images with AI-powered creativity',
					specs: 'n8n Workflow • Image Generation',
					tags: ['Creative', 'Workflow']
				};
			}

			// Default pipeline
			return {
				emoji: '🔗',
				name: model.name || modelId.replace(/[_.:]/g, ' '),
				description: 'Pipeline for advanced processing workflows',
				specs: 'Pipeline • Multi-step processing',
				tags: ['Workflow']
			};
		}

		// Handle functions/pipelines by ID pattern matching (catch all for items without type)
		if (modelId.toLowerCase().includes('marketing') || modelId === 'marketingCode') {
			return {
				emoji: '📊',
				name: 'Marketing Code',
				description: 'Reviews campaign concepts and visuals against the Allwyn Marketing Code',
				specs: 'Brand compliance • Campaign review',
				tags: ['AI Chat', 'Marketing']
			};
		}

		if (modelId.toLowerCase().includes('audio') || modelId === 'audioGen') {
			return {
				emoji: '🎵',
				name: 'Audio Generator',
				description: 'Create high-quality audio content from text scripts or prompts',
				specs: 'Conversational • Audio Synthesis',
				tags: ['Audio', 'Creative']
			};
		}

		if (modelId.toLowerCase().includes('video') || modelId === 'videogen') {
			return {
				emoji: '🎬',
				name: 'Video Generator',
				description: 'Generate, edit and animate your videos with AI-powered creativity',
				specs: 'n8n Workflow • Video Generation',
				tags: ['Creative', 'Workflow']
			};
		}

		if (modelId.toLowerCase().includes('image') || modelId === 'imageGen' || modelId.includes('n8n')) {
			return {
				emoji: '🎨',
				name: 'Image Generator',
				description: 'Generate, edit and animate your images with AI-powered creativity',
				specs: 'n8n Workflow • Image Generation',
				tags: ['Creative', 'Workflow']
			};
		}

		if (modelId.toLowerCase().includes('researcher')) {
			return {
				emoji: '🔍',
				name: 'User Researcher',
				description: 'Expert user researcher for fast, unmoderated testing via self-serve research platforms',
				specs: 'Research focused • User testing',
				tags: ['Research']
			};
		}

		// Handle specific AI models by ID patterns
		if (modelId.includes('qwen')) {
			return {
				emoji: '⚡',
				name: 'Qwen',
				description: 'Reasoning chat with fast responses',
				specs: '8B parameters • Fast inference',
				tags: ['AI Chat']
			};
		}

		if (modelId.includes('mistral')) {
			return {
				emoji: '🌟',
				name: 'Mistral',
				description: 'Efficient and versatile model for general purpose tasks',
				specs: '7B parameters • Balanced performance',
				tags: ['AI Chat']
			};
		}

		if (modelId.includes('vision')) {
			return {
				emoji: '👀',
				name: 'Vision',
				description: 'Advanced vision model for image analysis and visual Q&A',
				specs: '11B parameters • Vision capable',
				tags: ['Vision', 'AI Chat']
			};
		}

		if (modelId.includes('arena')) {
			return {
				emoji: '🏟️',
				name: 'Arena Model',
				description: 'Specialized model for advanced reasoning and complex tasks',
				specs: 'Arena-based • Advanced reasoning',
				tags: ['AI Chat']
			};
		}

		// Default for regular models
		return {
			emoji: '🤖',
			name: model?.name || modelId.replace(/[_.:]/g, ' '),
			description: 'AI language model',
			specs: 'Available for chat',
			tags: ['AI Chat']
		};
	};
</script>

<div
	class="h-screen max-h-[100dvh] transition-width duration-200 ease-in-out {$showSidebar
		? 'md:max-w-[calc(100%-260px)]'
		: ''} w-full max-w-full flex flex-col"
>
	<!-- Hamburger menu button -->
	<div class="sticky top-0 z-30 w-full py-1.5">
		<div class="flex items-center w-full px-1.5">
			<div class="{$showSidebar ? 'md:hidden' : ''} mr-1 flex items-center">
				<button
					id="sidebar-toggle-button"
					class="cursor-pointer px-2 py-2 flex rounded-xl hover:bg-gray-50 dark:hover:bg-gray-850 transition text-gray-600 dark:text-gray-400"
					on:click={() => {
						showSidebar.set(!$showSidebar);
					}}
					aria-label="Toggle Sidebar"
				>
					<div class="m-auto self-center">
						<MenuLines />
					</div>
				</button>
			</div>
		</div>
	</div>

	<div class="landing-container">
		<div class="header-section">
			<img src="/static/logo.png" alt="Allwyn AI Studio" class="logo" />
			<h1 class="main-title">Welcome to Allwyn AI Studio</h1>
			<p class="subtitle">Choose your use case and start generating</p>
		</div>

	<!-- Filter and Search Section -->
	<div class="filters-section">
		<div class="search-container">
			<input
				type="text"
				bind:value={searchQuery}
				placeholder="Search models..."
				class="search-input"
			/>
		</div>

		<div class="filter-chips">
			<button
				class="filter-chip"
				class:active={selectedTags.length === 0 && searchQuery === ''}
				on:click={clearAllFilters}
			>
				All
			</button>
			{#each allTags as tag}
				<button
					class="filter-chip"
					class:active={selectedTags.includes(tag)}
					on:click={() => toggleTag(tag)}
				>
					{tag}
				</button>
			{/each}
		</div>
	</div>

	<div class="models-grid">
		{#each filteredModels as model (model.id)}
			{@const modelInfo = getModelInfo(model.id)}
			<button
				class="model-card"
				on:click={() => startChatWithModel(model.id)}
			>
				<div class="model-emoji">{modelInfo.emoji}</div>
				<div class="model-name">{modelInfo.name}</div>
				<div class="model-description">{modelInfo.description}</div>
				<div class="model-specs">{modelInfo.specs}</div>
			</button>
		{/each}

		{#if filteredModels.length === 0}
			<div class="no-results">
				<div class="no-results-emoji">🔍</div>
				<div class="no-results-text">No models found</div>
				<div class="no-results-subtext">Try adjusting your filters or search query</div>
			</div>
		{/if}
	</div>

		<div class="footer">
			<p>Vibe coded with love ❤️</p>
		</div>
	</div>
</div>

<style>
	.landing-container {
		height: 100%;
		padding: 3.2rem;
		color: var(--color-gray-900);
		background: var(--color-gray-50);
		overflow-y: auto;
	}

	/* Dark mode colors */
	:global(.dark) .landing-container {
		color: var(--color-gray-100);
		background: var(--color-gray-900);
	}

	.header-section {
		text-align: center;
		margin: 0 auto 3rem auto;
		max-width: 800px;
	}

	.logo {
		width: 64px;
		height: 64px;
		margin: 0 auto 1rem auto;
		border-radius: 666px;
		display: block;
	}

	.main-title {
		font-size: 2.2rem;
		font-weight: 600;
		color: var(--color-gray-900);
	}

	:global(.dark) .main-title {
		color: var(--color-gray-100);
	}

	.subtitle {
		font-size: 1rem;
		font-weight: 400;
		opacity: 0.8;
		margin-bottom: 0;
		color: var(--color-gray-600);
	}

	:global(.dark) .subtitle {
		color: var(--color-gray-400);
	}

	.filters-section {
		max-width: 1200px;
		margin: 0 auto 2rem auto;
	}

	.search-container {
		margin-bottom: 1.5rem;
		display: flex;
		justify-content: center;
	}

	.search-input {
		width: 100%;
		max-width: 400px;
		padding: 0.75rem 1rem;
		background: rgba(var(--color-gray-900-rgb, 23, 23, 23), 0.05);
		backdrop-filter: blur(10px);
		border: 1px solid rgba(var(--color-gray-900-rgb, 23, 23, 23), 0.1);
		border-radius: 12px;
		color: var(--color-gray-900);
		font-size: 0.95rem;
		outline: none;
		transition: all 0.3s ease;
	}

	:global(.dark) .search-input {
		background: rgba(var(--color-gray-100-rgb, 236, 236, 236), 0.1);
		border-color: rgba(var(--color-gray-100-rgb, 236, 236, 236), 0.2);
		color: var(--color-gray-100);
	}

	.search-input::placeholder {
		color: var(--color-gray-500);
	}

	:global(.dark) .search-input::placeholder {
		color: var(--color-gray-400);
	}

	.search-input:focus {
		border-color: rgba(var(--color-gray-900-rgb, 23, 23, 23), 0.2);
		background: rgba(var(--color-gray-900-rgb, 23, 23, 23), 0.08);
		box-shadow: 0 0 20px rgba(var(--color-gray-900-rgb, 23, 23, 23), 0.05);
	}

	:global(.dark) .search-input:focus {
		border-color: rgba(var(--color-gray-100-rgb, 236, 236, 236), 0.4);
		background: rgba(var(--color-gray-100-rgb, 236, 236, 236), 0.15);
		box-shadow: 0 0 20px rgba(var(--color-gray-100-rgb, 236, 236, 236), 0.1);
	}

	.filter-chips {
		display: flex;
		flex-wrap: wrap;
		gap: 0.5rem;
		justify-content: center;
		align-items: center;
	}

	.filter-chip {
		padding: 0.5rem 1rem;
		background: rgba(var(--color-gray-900-rgb, 23, 23, 23), 0.05);
		backdrop-filter: blur(10px);
		border: 1px solid rgba(var(--color-gray-900-rgb, 23, 23, 23), 0.1);
		border-radius: 20px;
		color: var(--color-gray-700);
		font-size: 0.85rem;
		font-weight: 500;
		cursor: pointer;
		transition: all 0.3s ease;
		outline: none;
	}

	:global(.dark) .filter-chip {
		background: rgba(var(--color-gray-100-rgb, 236, 236, 236), 0.1);
		border-color: rgba(var(--color-gray-100-rgb, 236, 236, 236), 0.2);
		color: var(--color-gray-300);
	}

	.filter-chip:hover {
		background: rgba(var(--color-gray-900-rgb, 23, 23, 23), 0.08);
		border-color: rgba(var(--color-gray-900-rgb, 23, 23, 23), 0.15);
		color: var(--color-gray-900);
		transform: translateY(-1px);
	}

	:global(.dark) .filter-chip:hover {
		background: rgba(var(--color-gray-100-rgb, 236, 236, 236), 0.15);
		border-color: rgba(var(--color-gray-100-rgb, 236, 236, 236), 0.3);
		color: var(--color-gray-100);
	}

	.filter-chip.active {
		background: rgba(var(--color-gray-900-rgb, 23, 23, 23), 0.12);
		border-color: rgba(var(--color-gray-900-rgb, 23, 23, 23), 0.2);
		color: var(--color-gray-900);
		font-weight: 600;
	}

	:global(.dark) .filter-chip.active {
		background: rgba(var(--color-gray-100-rgb, 236, 236, 236), 0.25);
		border-color: rgba(var(--color-gray-100-rgb, 236, 236, 236), 0.4);
		color: var(--color-gray-100);
	}

	.no-results {
		grid-column: 1 / -1;
		text-align: center;
		padding: 3rem 2rem;
		color: var(--color-gray-500);
	}

	:global(.dark) .no-results {
		color: var(--color-gray-400);
	}

	.no-results-emoji {
		font-size: 3rem;
		margin-bottom: 1rem;
		opacity: 0.5;
	}

	.no-results-text {
		font-size: 1.2rem;
		font-weight: 600;
		margin-bottom: 0.5rem;
		color: var(--color-gray-700);
	}

	:global(.dark) .no-results-text {
		color: var(--color-gray-300);
	}

	.no-results-subtext {
		font-size: 0.9rem;
		opacity: 0.7;
	}

	.models-grid {
		display: grid;
		grid-template-columns: repeat(auto-fit, minmax(320px, 1fr));
		gap: 1.48rem;
		max-width: 1200px;
		margin: 0 auto;
	}

	.model-card {
		background: rgba(var(--color-gray-900-rgb, 23, 23, 23), 0.05);
		backdrop-filter: blur(10px);
		border: 1px solid rgba(var(--color-gray-900-rgb, 23, 23, 23), 0.08);
		border-radius: 16px;
		padding: 1rem 2.4rem;
		color: var(--color-gray-900);
		text-align: center;
		cursor: pointer;
		transition: all 0.3s ease;
		outline: none;
	}

	:global(.dark) .model-card {
		background: rgba(var(--color-gray-100-rgb, 236, 236, 236), 0.1);
		border-color: rgba(var(--color-gray-100-rgb, 236, 236, 236), 0.15);
		color: var(--color-gray-100);
	}

	.model-card:hover {
		transform: translateY(-5px);
		background: rgba(var(--color-gray-900-rgb, 23, 23, 23), 0.08);
		border-color: rgba(var(--color-gray-900-rgb, 23, 23, 23), 0.12);
		box-shadow: 0 20px 40px rgba(var(--color-gray-900-rgb, 23, 23, 23), 0.1);
	}

	:global(.dark) .model-card:hover {
		background: rgba(var(--color-gray-100-rgb, 236, 236, 236), 0.2);
		border-color: rgba(var(--color-gray-100-rgb, 236, 236, 236), 0.25);
		box-shadow: 0 20px 40px rgba(0, 0, 0, 0.3);
	}

	.model-card:focus {
		outline: 2px solid var(--color-gray-400);
		outline-offset: 2px;
	}

	:global(.dark) .model-card:focus {
		outline-color: var(--color-gray-500);
	}

	.model-emoji {
		font-size: 2.48rem;
		margin-bottom: 1rem;
	}

	.model-name {
		font-size: 1.32rem;
		font-weight: 600;
		margin-bottom: 0.24rem;
		color: var(--color-gray-900);
	}

	:global(.dark) .model-name {
		color: var(--color-gray-100);
	}

	.model-description {
		font-size: 0.92rem;
		opacity: 0.7;
		line-height: 1.5;
		margin-bottom: 1.24rem;
		color: var(--color-gray-600);
	}

	:global(.dark) .model-description {
		color: var(--color-gray-400);
	}

	.model-specs {
		font-size: 0.72rem;
		opacity: 0.6;
		font-style: italic;
		color: var(--color-gray-500);
	}

	:global(.dark) .model-specs {
		color: var(--color-gray-500);
	}

	.footer {
		text-align: center;
		margin-top: 4rem;
		padding: 2rem;
		opacity: 0.6;
	}

	.footer p {
		font-size: 0.9rem;
		font-weight: 400;
		color: var(--color-gray-600);
		margin: 0;
	}

	:global(.dark) .footer p {
		color: var(--color-gray-400);
	}

	@media (max-width: 768px) {
		.landing-container {
			padding: 1rem;
			width: 100vw;
		}

		.main-title {
			font-size: 2rem;
		}

		.filters-section {
			margin-bottom: 1.5rem;
		}

		.search-input {
			max-width: 100%;
			font-size: 16px; /* Prevents zoom on iOS */
		}

		.filter-chips {
			gap: 0.375rem;
		}

		.filter-chip {
			padding: 0.375rem 0.75rem;
			font-size: 0.8rem;
		}

		.models-grid {
			grid-template-columns: 1fr;
			gap: 1rem;
		}

		.model-card {
			padding: 1.5rem;
		}
	}

	/* System theme preference support */
	@media (prefers-color-scheme: light) {
		:root:not(.dark) .landing-container {
			color: var(--color-gray-900);
			background: var(--color-gray-50);
		}
	}

	@media (prefers-color-scheme: dark) {
		:root:not(.light) .landing-container {
			color: var(--color-gray-100);
			background: var(--color-gray-900);
		}
	}
</style>