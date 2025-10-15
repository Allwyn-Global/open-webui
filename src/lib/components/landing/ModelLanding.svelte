<script lang="ts">
	import { getContext, onMount } from 'svelte';
	import { goto } from '$app/navigation';
	import { models, showSidebar } from '$lib/stores';
	import { getModels, getPipelines } from '$lib/apis';
	import { getFunctions } from '$lib/apis/functions';
	import MenuLines from '../icons/MenuLines.svelte';
	import Search from '../icons/Search.svelte';
	import type { i18n as i18nType } from 'i18next';
	const i18n: Writable<i18nType> = getContext('i18n');
	import type { Writable } from 'svelte/store';

	let availableModels = [];
	let filteredModels = [];
	let selectedTags = [];
	let searchQuery = '';
	let searchExpanded = false;

	// Get current year for copyright
	const currentYear = new Date().getFullYear();

	// Get all unique tags from dynamically loaded models, excluding 'Workflow'
	$: allTags = [...new Set(filteredModels.map(model => getModelInfo(model.id).tags || []).flat())]
		.filter(tag => tag !== 'Workflow')
		.sort();

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

	// Auto-focus input when expanded
	const focusOnMount = (node: HTMLInputElement) => {
		node.focus();
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

		// Extract name and description from model metadata
		const getFullName = () => {
			// Try to get name from model metadata
			if (model?.name) return model.name;
			if (model?.info?.name) return model.info.name;
			// Fallback: format ID as name
			return modelId.replace(/[_.:]/g, ' ').replace(/\b\w/g, l => l.toUpperCase());
		};

		// Extract emoji from name or use default
		const fullName = getFullName();
		const emojiRegex = /^(\p{Emoji_Presentation}|\p{Emoji}\uFE0F)/u;
		const emojiMatch = fullName.match(emojiRegex);

		const getEmoji = () => {
			// If name starts with emoji, use it
			if (emojiMatch) {
				return emojiMatch[0];
			}

			// Otherwise use pattern-based fallback
			const type = model?.type;
			if (type === 'function') return '⚙️';
			if (type === 'pipeline') return '🔗';

			// Pattern matching on ID/name
			const searchText = (fullName + modelId).toLowerCase();
			if (searchText.includes('video')) return '🎬';
			if (searchText.includes('image') || searchText.includes('n8n')) return '🎨';
			if (searchText.includes('audio')) return '🎵';
			if (searchText.includes('marketing')) return '📊';
			if (searchText.includes('research')) return '🔍';
			if (searchText.includes('vision')) return '👀';
			return '🤖';
		};

		const getName = () => {
			// Remove emoji from name if present
			if (emojiMatch) {
				return fullName.slice(emojiMatch[0].length).trim();
			}
			return fullName;
		};

		const getDescription = () => {
			// Try to get description from model metadata
			if (model?.meta?.description) return model.meta.description;
			if (model?.info?.meta?.description) return model.info.meta.description;
			if (model?.description) return model.description;
			// Fallback generic descriptions
			if (model?.type === 'function') return 'Custom function for enhanced capabilities';
			if (model?.type === 'pipeline') return 'Pipeline for advanced processing workflows';
			return 'AI language model';
		};

		// Get tags based on model type and ID patterns
		const getTags = () => {
			const tags = [];

			// If model has tags, extract string values
			if (model?.tags && Array.isArray(model.tags)) {
				// Handle both string arrays and object arrays
				const extractedTags = model.tags.map(tag => {
					if (typeof tag === 'string') return tag;
					if (tag?.name) return tag.name;
					if (tag?.label) return tag.label;
					return null;
				}).filter(tag => tag !== null);

				if (extractedTags.length > 0) return extractedTags;
			}

			// Infer tags from ID and type
			if (model?.type === 'function' || model?.type === 'pipeline') {
				if (modelId.toLowerCase().includes('video') || modelId.toLowerCase().includes('image') || modelId.toLowerCase().includes('audio')) {
					tags.push('Creative');
				}
				if (modelId.toLowerCase().includes('marketing')) {
					tags.push('Marketing');
				}
				if (modelId.toLowerCase().includes('research')) {
					tags.push('Research');
				}
			} else {
				tags.push('AI Chat');
			}

			return tags.length > 0 ? tags : ['AI Chat'];
		};

		// Use dynamic data with emoji extraction
		return {
			emoji: getEmoji(),
			name: getName(),
			description: getDescription(),
			tags: getTags()
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
		<div class="filters-row">
			{#if !searchExpanded}
				<!-- Filter Tags (shown when search is collapsed) -->
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

				<!-- Search Icon (at the end) -->
				<button
					class="search-icon-button"
					on:click={() => searchExpanded = true}
					aria-label="Search"
				>
					<Search className="w-5 h-5" />
				</button>
			{:else}
				<!-- Expanded Search Input (replaces filters) -->
				<div class="search-input-wrapper">
					<input
						type="text"
						bind:value={searchQuery}
						placeholder="Search models..."
						class="search-input"
						on:blur={() => {
							if (!searchQuery) {
								searchExpanded = false;
							}
						}}
						use:focusOnMount
					/>
					{#if searchQuery}
						<button
							class="search-clear-button"
							on:click={() => {
								searchQuery = '';
								searchExpanded = false;
							}}
							aria-label="Clear search"
						>
							✕
						</button>
					{/if}
				</div>
			{/if}
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
				<div class="model-content">
					<div class="model-name">{modelInfo.name}</div>
					<div class="model-description">{modelInfo.description}</div>
				</div>
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
			<p>© {currentYear} Allwyn, All rights reserved</p>
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

	.filters-row {
		display: flex;
		align-items: center;
		gap: 1rem;
		justify-content: center;
		flex-wrap: wrap;
	}

	.search-container {
		display: flex;
		align-items: center;
	}

	.search-icon-button {
		padding: 0.5rem;
		background: rgba(var(--color-gray-900-rgb, 23, 23, 23), 0.05);
		backdrop-filter: blur(10px);
		border: 1px solid rgba(var(--color-gray-900-rgb, 23, 23, 23), 0.1);
		border-radius: 9999px; /* Fully rounded */
		color: var(--color-gray-700);
		cursor: pointer;
		transition: all 0.3s ease;
		outline: none;
		display: flex;
		align-items: center;
		justify-content: center;
		flex-shrink: 0;
	}

	:global(.dark) .search-icon-button {
		background: rgba(var(--color-gray-100-rgb, 236, 236, 236), 0.1);
		border-color: rgba(var(--color-gray-100-rgb, 236, 236, 236), 0.2);
		color: var(--color-gray-300);
	}

	.search-icon-button:hover {
		background: rgba(var(--color-gray-900-rgb, 23, 23, 23), 0.08);
		border-color: rgba(var(--color-gray-900-rgb, 23, 23, 23), 0.15);
		color: var(--color-gray-900);
		transform: translateY(-1px);
	}

	:global(.dark) .search-icon-button:hover {
		background: rgba(var(--color-gray-100-rgb, 236, 236, 236), 0.15);
		border-color: rgba(var(--color-gray-100-rgb, 236, 236, 236), 0.3);
		color: var(--color-gray-100);
	}

	.search-input-wrapper {
		position: relative;
		display: flex;
		align-items: center;
		flex: 1;
		animation: expandSearch 0.3s ease;
	}

	@keyframes expandSearch {
		from {
			opacity: 0;
			transform: scaleX(0.8);
		}
		to {
			opacity: 1;
			transform: scaleX(1);
		}
	}

	.search-input {
		flex: 1;
		width: 100%;
		padding: 0.5rem 2.5rem 0.5rem 1rem;
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

	.search-clear-button {
		position: absolute;
		right: 0.75rem;
		background: none;
		border: none;
		color: var(--color-gray-500);
		cursor: pointer;
		padding: 0.25rem;
		display: flex;
		align-items: center;
		justify-content: center;
		font-size: 1rem;
		transition: color 0.2s ease;
	}

	:global(.dark) .search-clear-button {
		color: var(--color-gray-400);
	}

	.search-clear-button:hover {
		color: var(--color-gray-900);
	}

	:global(.dark) .search-clear-button:hover {
		color: var(--color-gray-100);
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
		background: #27E2CC;
		border-color: #27E2CC;
		color: #000;
		font-weight: 600;
	}

	:global(.dark) .filter-chip.active {
		background: #27E2CC;
		border-color: #27E2CC;
		color: #000;
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
		grid-template-columns: repeat(auto-fit, minmax(280px, 1fr));
		gap: 1rem;
		max-width: 1200px;
		margin: 0 auto;
	}

	.model-card {
		background: rgba(var(--color-gray-900-rgb, 23, 23, 23), 0.05);
		backdrop-filter: blur(10px);
		border: 1px solid rgba(var(--color-gray-900-rgb, 23, 23, 23), 0.08);
		border-radius: 12px;
		padding: 1rem 1.25rem;
		color: var(--color-gray-900);
		text-align: left;
		cursor: pointer;
		transition: all 0.3s ease;
		outline: none;
		display: flex;
		align-items: center;
		gap: 1rem;
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
		font-size: 1.75rem;
		flex-shrink: 0;
		display: flex;
		align-items: center;
		justify-content: center;
	}

	.model-content {
		flex: 1;
		min-width: 0;
	}

	.model-name {
		font-size: 1.05rem;
		font-weight: 600;
		margin-bottom: 0.3rem;
		color: var(--color-gray-900);
		line-height: 1.3;
	}

	:global(.dark) .model-name {
		color: var(--color-gray-100);
	}

	.model-description {
		font-size: 0.8rem;
		opacity: 0.7;
		line-height: 1.4;
		margin-bottom: 0;
		color: var(--color-gray-600);
	}

	:global(.dark) .model-description {
		color: var(--color-gray-400);
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

		.filters-row {
			flex-direction: column;
			gap: 0.75rem;
		}

		.search-input {
			width: 100%;
			max-width: 100%;
			font-size: 16px; /* Prevents zoom on iOS */
		}

		.search-input-wrapper {
			width: 100%;
		}

		@keyframes expandSearch {
			from {
				width: 40px;
				opacity: 0;
			}
			to {
				width: 100%;
				opacity: 1;
			}
		}

		.filter-chips {
			gap: 0.375rem;
			width: 100%;
		}

		.filter-chip {
			padding: 0.375rem 0.75rem;
			font-size: 0.8rem;
		}

		.models-grid {
			grid-template-columns: 1fr;
			gap: 0.75rem;
		}

		.model-card {
			padding: 0.875rem 1rem;
			gap: 0.875rem;
		}

		.model-emoji {
			font-size: 1.5rem;
		}

		.model-name {
			font-size: 0.95rem;
		}

		.model-description {
			font-size: 0.75rem;
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