<script lang="ts">
	import { goto } from '$app/navigation';
	import { getContext, onMount } from 'svelte';

	const i18n = getContext('i18n');

	let hasScript = 'yes';
	let scriptText = '';
	let brief = '';
	let length = '30';
	let language = 'English';
	let voice = 'Auto';
	let isSubmitting = false;

	// Voice options
	const voices = [
		{ name: 'Auto', description: 'Automatically determined based on the script' },
		{ name: 'Aria', description: 'Female, American accent, clear and expressive (good for narration)' },
		{ name: 'Roger', description: 'Male, American accent, warm and neutral' },
		{ name: 'Sarah', description: 'Female, American accent, soft and news-style' },
		{ name: 'Laura', description: 'Female, American accent, steady and approachable' },
		{ name: 'Charlie', description: 'Male, Australian accent, energetic and conversational' },
		{ name: 'George', description: 'Male, British accent, formal and authoritative' },
		{ name: 'Callum', description: 'Male, American accent, casual and friendly' },
		{ name: 'River', description: 'Versatile, neutral tone' },
		{ name: 'Liam', description: 'Male, American accent, youthful and clear' },
		{ name: 'Charlotte', description: 'Female, Swedish accent, calm and measured' },
		{ name: 'Alice', description: 'Female, British accent, elegant and soft' },
		{ name: 'Matilda', description: 'Female, American accent, warm and narrative' },
		{ name: 'Will', description: 'Male, American accent, friendly and social-media tone' },
		{ name: 'Jessica', description: 'Female, American accent, conversational and bright' },
		{ name: 'Eric', description: 'Male, American accent, clear and solid' },
		{ name: 'Chris', description: 'Male, American accent, neutral and adaptable' },
		{ name: 'Brian', description: 'Male, American accent, steady and calm' },
		{ name: 'Daniel', description: 'Male, British accent, composed and articulate' },
		{ name: 'Lily', description: 'Female, British accent, gentle and friendly' },
		{ name: 'Bill', description: 'Male, American accent, multilingual, consistent style across languages' }
	];

	// Clear any stale audiogen message when form loads
	onMount(() => {
		sessionStorage.removeItem('audiogen_initial_message');
	});

	// Extract first number from string (supports digits or words one-nine)
	const extractLength = (input: string): number => {
		const wordToNumber: { [key: string]: number } = {
			'one': 1, 'two': 2, 'three': 3, 'four': 4, 'five': 5,
			'six': 6, 'seven': 7, 'eight': 8, 'nine': 9
		};

		// Try to find a number (digits)
		const digitMatch = input.match(/\d+/);
		if (digitMatch) {
			const num = parseInt(digitMatch[0]);
			return Math.min(Math.max(num, 1), 60); // Clamp between 1-60
		}

		// Try to find word numbers
		const words = input.toLowerCase().split(/\s+/);
		for (const word of words) {
			if (wordToNumber[word]) {
				return wordToNumber[word];
			}
		}

		// Default to 30 if no number found
		return 30;
	};

	const handleSubmit = async () => {
		// Validate based on hasScript choice
		if (hasScript === 'yes' && !scriptText.trim()) {
			return;
		}
		if (hasScript === 'no' && !brief.trim()) {
			return;
		}

		isSubmitting = true;

		let formattedMessage = '';

		if (hasScript === 'yes') {
			// User has a script
			formattedMessage = `Has Script: Yes
Script: ${scriptText.trim()}
Voice: ${voice}`;
		} else {
			// User doesn't have a script
			const extractedLength = extractLength(length);
			formattedMessage = `Has Script: No
Brief: ${brief.trim()}
Length: ${extractedLength}s
Language: ${language}
Voice: ${voice}`;
		}

		console.log('=== AUDIOGEN FORM SUBMIT DEBUG ===');
		console.log('Formatted message:', formattedMessage);

		// Clear any stored model selections to ensure URL parameter is used
		sessionStorage.removeItem('selectedModels');
		console.log('Cleared sessionStorage.selectedModels');

		// Store the message in sessionStorage to be picked up by the chat interface
		sessionStorage.setItem('audiogen_initial_message', formattedMessage);
		console.log('Stored audiogen_initial_message in sessionStorage');

		// Navigate to chat with the audiogen model/pipeline
		const targetUrl = '/?model=audiogen';
		console.log('Navigating to:', targetUrl);
		console.log('=== END AUDIOGEN FORM SUBMIT DEBUG ===');

		await goto(targetUrl);
	};
</script>

<div class="flex flex-col flex-1 overflow-y-auto">
	<div class="max-w-3xl mx-auto w-full px-4 py-8">
		<div class="mb-6">
			<h1 class="text-2xl font-semibold text-gray-900 dark:text-white mb-2">
				Voice Generator
			</h1>
			<p class="text-sm text-gray-600 dark:text-gray-400">
				Generate audio content with or without a script
			</p>
		</div>

		<form on:submit|preventDefault={handleSubmit} class="space-y-6">
			<!-- Script Availability Question -->
			<div>
				<label class="block text-sm font-medium text-gray-900 dark:text-white mb-3">
					Do you have a script to use? <span class="text-red-500">*</span>
				</label>
				<div class="space-y-2">
					<label class="flex items-center cursor-pointer">
						<input
							type="radio"
							bind:group={hasScript}
							value="yes"
							class="w-4 h-4 text-gray-900 dark:text-white border-gray-300 dark:border-gray-600 focus:ring-2 focus:ring-gray-500 dark:focus:ring-gray-400"
						/>
						<span class="ml-3 text-sm text-gray-900 dark:text-white">
							Yes, I have a script ready
						</span>
					</label>
					<label class="flex items-center cursor-pointer">
						<input
							type="radio"
							bind:group={hasScript}
							value="no"
							class="w-4 h-4 text-gray-900 dark:text-white border-gray-300 dark:border-gray-600 focus:ring-2 focus:ring-gray-500 dark:focus:ring-gray-400"
						/>
						<span class="ml-3 text-sm text-gray-900 dark:text-white">
							No, I'll provide a brief
						</span>
					</label>
				</div>
			</div>

			{#if hasScript === 'yes'}
				<!-- Script Text Input -->
				<div>
					<label for="script" class="block text-sm font-medium text-gray-900 dark:text-white mb-2">
						Script Text <span class="text-red-500">*</span>
					</label>
					<textarea
						id="script"
						bind:value={scriptText}
						required
						rows="8"
						class="w-full rounded-lg border border-gray-300 dark:border-gray-600 bg-white dark:bg-gray-800 px-4 py-3 text-sm text-gray-900 dark:text-white placeholder-gray-500 dark:placeholder-gray-400 focus:border-gray-500 dark:focus:border-gray-400 focus:outline-none resize-y"
						placeholder="Enter your script here..."
					/>
					<p class="mt-2 text-xs text-gray-500 dark:text-gray-400">
						Provide the exact text you want to be spoken
					</p>
				</div>

				<!-- Voice Select (for hasScript = yes) -->
				<div>
					<label for="voice-yes" class="block text-sm font-medium text-gray-900 dark:text-white mb-2">
						Voice <span class="text-red-500">*</span>
					</label>
					<select
						id="voice-yes"
						bind:value={voice}
						class="w-full rounded-lg border border-gray-300 dark:border-gray-600 bg-white dark:bg-gray-800 px-4 py-3 text-sm text-gray-900 dark:text-white focus:border-gray-500 dark:focus:border-gray-400 focus:outline-none"
					>
						{#each voices as voiceOption}
							<option value={voiceOption.name}>
								{voiceOption.name} - {voiceOption.description}
							</option>
						{/each}
					</select>
					<p class="mt-2 text-xs text-gray-500 dark:text-gray-400">
						Select a voice or use Auto to determine automatically
					</p>
				</div>
			{:else}
				<!-- Brief Input -->
				<div>
					<label for="brief" class="block text-sm font-medium text-gray-900 dark:text-white mb-2">
						Brief <span class="text-red-500">*</span>
					</label>
					<textarea
						id="brief"
						bind:value={brief}
						required
						rows="6"
						class="w-full rounded-lg border border-gray-300 dark:border-gray-600 bg-white dark:bg-gray-800 px-4 py-3 text-sm text-gray-900 dark:text-white placeholder-gray-500 dark:placeholder-gray-400 focus:border-gray-500 dark:focus:border-gray-400 focus:outline-none resize-y"
						placeholder="Describe what the audio should be about..."
					/>
					<p class="mt-2 text-xs text-gray-500 dark:text-gray-400">
						Provide a detailed description of what you want the audio to say
					</p>
				</div>

				<!-- Length Input -->
				<div>
					<label for="length" class="block text-sm font-medium text-gray-900 dark:text-white mb-2">
						Length (seconds) <span class="text-red-500">*</span>
					</label>
					<input
						id="length"
						type="text"
						bind:value={length}
						required
						class="w-full rounded-lg border border-gray-300 dark:border-gray-600 bg-white dark:bg-gray-800 px-4 py-3 text-sm text-gray-900 dark:text-white placeholder-gray-500 dark:placeholder-gray-400 focus:border-gray-500 dark:focus:border-gray-400 focus:outline-none"
						placeholder="e.g., 30 or thirty"
					/>
					<p class="mt-2 text-xs text-gray-500 dark:text-gray-400">
						Enter a number between 1-60 (as digits or words like "thirty")
					</p>
				</div>

				<!-- Language Select -->
				<div>
					<label for="language" class="block text-sm font-medium text-gray-900 dark:text-white mb-2">
						Language <span class="text-red-500">*</span>
					</label>
					<select
						id="language"
						bind:value={language}
						class="w-full rounded-lg border border-gray-300 dark:border-gray-600 bg-white dark:bg-gray-800 px-4 py-3 text-sm text-gray-900 dark:text-white focus:border-gray-500 dark:focus:border-gray-400 focus:outline-none"
					>
						<option value="English">English</option>
						<option value="German">German</option>
						<option value="Czech">Czech</option>
						<option value="Greek">Greek</option>
					</select>
				</div>

				<!-- Voice Select (for hasScript = no) -->
				<div>
					<label for="voice-no" class="block text-sm font-medium text-gray-900 dark:text-white mb-2">
						Voice <span class="text-red-500">*</span>
					</label>
					<select
						id="voice-no"
						bind:value={voice}
						class="w-full rounded-lg border border-gray-300 dark:border-gray-600 bg-white dark:bg-gray-800 px-4 py-3 text-sm text-gray-900 dark:text-white focus:border-gray-500 dark:focus:border-gray-400 focus:outline-none"
					>
						{#each voices as voiceOption}
							<option value={voiceOption.name}>
								{voiceOption.name} - {voiceOption.description}
							</option>
						{/each}
					</select>
					<p class="mt-2 text-xs text-gray-500 dark:text-gray-400">
						Select a voice or use Auto to determine automatically
					</p>
				</div>
			{/if}

			<!-- Submit Button -->
			<div class="flex justify-end pt-4">
				<button
					type="submit"
					disabled={isSubmitting || (hasScript === 'yes' ? !scriptText.trim() : !brief.trim())}
					class="px-6 py-3 rounded-lg bg-gray-900 dark:bg-white text-white dark:text-gray-900 font-medium text-sm hover:bg-gray-800 dark:hover:bg-gray-100 disabled:opacity-50 disabled:cursor-not-allowed transition-colors"
				>
					{isSubmitting ? 'Generating...' : 'Generate Audio'}
				</button>
			</div>
		</form>
	</div>
</div>
