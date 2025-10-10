<script lang="ts">
	import { goto } from '$app/navigation';
	import { getContext } from 'svelte';

	const i18n = getContext('i18n');

	let script = '';
	let duration = '12';
	let aspectRatio = 'landscape';
	let isSubmitting = false;

	const handleSubmit = async () => {
		// Validate script is not empty
		if (!script.trim()) {
			return;
		}

		isSubmitting = true;

		// Format the message for the pipeline
		const formattedMessage = `Script: ${script.trim()}
Duration: ${duration}s
Aspect Ratio: ${aspectRatio}`;

		// Store the message in sessionStorage to be picked up by the chat interface
		sessionStorage.setItem('videogen_initial_message', formattedMessage);

		// Navigate to chat with the videogen model/pipeline
		// The chat interface will check for the stored message and send it automatically
		await goto('/?model=imageGen');
	};
</script>

<div class="flex flex-col flex-1 overflow-y-auto">
	<div class="max-w-3xl mx-auto w-full px-4 py-8">
		<div class="mb-6">
			<h1 class="text-2xl font-semibold text-gray-900 dark:text-white mb-2">
				Video Generator
			</h1>
			<p class="text-sm text-gray-600 dark:text-gray-400">
				Describe your video concept and configure the generation parameters
			</p>
		</div>

		<form on:submit|preventDefault={handleSubmit} class="space-y-6">
			<!-- Script Input -->
			<div>
				<label for="script" class="block text-sm font-medium text-gray-900 dark:text-white mb-2">
					Video Script <span class="text-red-500">*</span>
				</label>
				<textarea
					id="script"
					bind:value={script}
					required
					rows="6"
					class="w-full rounded-lg border border-gray-300 dark:border-gray-600 bg-white dark:bg-gray-800 px-4 py-3 text-sm text-gray-900 dark:text-white placeholder-gray-500 dark:placeholder-gray-400 focus:border-gray-500 dark:focus:border-gray-400 focus:outline-none resize-y"
					placeholder="Describe what you want to see in your video..."
				/>
				<p class="mt-2 text-xs text-gray-500 dark:text-gray-400">
					Provide a detailed description of your video concept
				</p>
			</div>

			<!-- Duration Select -->
			<div>
				<label for="duration" class="block text-sm font-medium text-gray-900 dark:text-white mb-2">
					Duration
				</label>
				<select
					id="duration"
					bind:value={duration}
					class="w-full rounded-lg border border-gray-300 dark:border-gray-600 bg-white dark:bg-gray-800 px-4 py-3 text-sm text-gray-900 dark:text-white focus:border-gray-500 dark:focus:border-gray-400 focus:outline-none"
				>
					<option value="4">4 seconds</option>
					<option value="8">8 seconds</option>
					<option value="12">12 seconds</option>
				</select>
			</div>

			<!-- Aspect Ratio Radio Buttons -->
			<div>
				<label class="block text-sm font-medium text-gray-900 dark:text-white mb-3">
					Aspect Ratio
				</label>
				<div class="space-y-2">
					<label class="flex items-center cursor-pointer">
						<input
							type="radio"
							bind:group={aspectRatio}
							value="landscape"
							class="w-4 h-4 text-gray-900 dark:text-white border-gray-300 dark:border-gray-600 focus:ring-2 focus:ring-gray-500 dark:focus:ring-gray-400"
						/>
						<span class="ml-3 text-sm text-gray-900 dark:text-white">
							Landscape (16:9)
						</span>
					</label>
					<label class="flex items-center cursor-pointer">
						<input
							type="radio"
							bind:group={aspectRatio}
							value="portrait"
							class="w-4 h-4 text-gray-900 dark:text-white border-gray-300 dark:border-gray-600 focus:ring-2 focus:ring-gray-500 dark:focus:ring-gray-400"
						/>
						<span class="ml-3 text-sm text-gray-900 dark:text-white">
							Portrait (9:16)
						</span>
					</label>
				</div>
			</div>

			<!-- Submit Button -->
			<div class="flex justify-end pt-4">
				<button
					type="submit"
					disabled={!script.trim() || isSubmitting}
					class="px-6 py-3 rounded-lg bg-gray-900 dark:bg-white text-white dark:text-gray-900 font-medium text-sm hover:bg-gray-800 dark:hover:bg-gray-100 disabled:opacity-50 disabled:cursor-not-allowed transition-colors"
				>
					{isSubmitting ? 'Generating...' : 'Generate Video'}
				</button>
			</div>
		</form>
	</div>
</div>
