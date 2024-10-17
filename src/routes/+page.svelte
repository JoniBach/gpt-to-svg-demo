<script>
	import { writable } from 'svelte/store';
	import { env } from '$env/dynamic/public';

	// Extract API key and URL from environment variables
	const apiKey = env.PUBLIC_API_KEY;
	const apiUrl = env.PUBLIC_API_URL;

	// Stores to handle response data
	let responseMessage = writable('');
	let generatedPrompt = writable('');
	let thumbnailUrl = writable('');
	let imageDownloadUrl = writable('');
	let svgDownloadUrl = writable('');
	let gcodeDownloadUrl = writable('');
	let zipDownloadUrl = writable('');
	let progressMessage = writable('');
	let isLoading = writable(false);
	let errorDetails = writable('');
	let startTime = writable('');
	let finishTime = writable('');

	let conceptInput = '';
	let isDarkMode = writable(false);

	// Load dark mode preference from local storage
	if (typeof localStorage !== 'undefined') {
		const savedTheme = localStorage.getItem('theme');
		if (savedTheme === 'dark') {
			isDarkMode.set(true);
		}
	}

	const toggleTheme = () => {
		isDarkMode.update((value) => {
			const newValue = !value;
			// Save the user's preference in local storage
			if (typeof localStorage !== 'undefined') {
				localStorage.setItem('theme', newValue ? 'dark' : 'light');
			}
			return newValue;
		});
	};

	// Function to call the API
	const generateImage = async () => {
		isLoading.set(true);
		progressMessage.set('Generating image, please wait...');
		errorDetails.set('');
		startTime.set(new Date().toLocaleTimeString());
		finishTime.set('');

		try {
			progressMessage.set('Sending request to generate the image...');

			const response = await fetch(apiUrl, {
				method: 'POST',
				headers: {
					'Content-Type': 'application/json',
					'x-api-key': apiKey,
					accept: 'application/json'
				},
				body: JSON.stringify({ concept: conceptInput })
			});

			if (!response.ok) {
				const errorText = await response.text();
				console.error('API Error:', errorText);
				responseMessage.set('Error: ' + response.status + ' - ' + errorText);
				progressMessage.set('Failed to generate image. Please try again.');
				errorDetails.set(`Status Code: ${response.status}, Error Text: ${errorText}`);
				isLoading.set(false);
				startTime.set('');

				return;
			}

			progressMessage.set('Processing the response...');
			const data = await response.json();

			progressMessage.set('Updating the UI with the generated image links...');
			responseMessage.set(data.message);
			generatedPrompt.set(data.prompt);
			thumbnailUrl.set(data.thumbnail);
			imageDownloadUrl.set(data.image_download_url);
			svgDownloadUrl.set(data.svg_download_url);
			gcodeDownloadUrl.set(data.gcode_download_url);
			zipDownloadUrl.set(data.zip_download_url);

			progressMessage.set('Image generation complete!');
			finishTime.set(new Date().toLocaleTimeString());
			isLoading.set(false);
		} catch (error) {
			console.error('Network Error:', error);
			responseMessage.set('A network error occurred while generating the image.');
			progressMessage.set('Network error occurred. Please check your connection.');
			errorDetails.set(`Network Error: ${error.message}`);
			isLoading.set(false);
			startTime.set('');
		}
	};
</script>

<div class={$isDarkMode ? 'dark' : 'light'}>
	<div class="container">
		<header>
			<div class="header-content">
				<h1>Image Generation API</h1>
				<button class="theme-toggle" on:click={toggleTheme}>
					{$isDarkMode ? '🌞 Light Mode' : '🌙 Dark Mode'}
				</button>
			</div>
		</header>

		<main>
			<div class="input-section">
				<input
					type="text"
					placeholder="Enter your concept..."
					bind:value={conceptInput}
					class="concept-input"
					disabled={$isLoading}
				/>
				<button on:click={generateImage} class="generate-button" disabled={$isLoading}>
					{$isLoading ? 'Generating...' : 'Generate Image'}
				</button>
			</div>

			<!-- Loading Marker -->
			{#if $isLoading}
				<p class="loading">This could take a minute...</p>
			{/if}

			<div class="response-container">
				{#if $responseMessage}
					<h2>Response</h2>
					<p>
						{$responseMessage}
						{#if $startTime && $finishTime}
							in {Math.round(
								(new Date(`1970-01-01T${$finishTime}Z`) - new Date(`1970-01-01T${$startTime}Z`)) /
									1000
							)} seconds
						{/if}
					</p>
				{/if}

				{#if $errorDetails}
					<div class="error-details">
						<p><strong>Error Details:</strong> {$errorDetails}</p>
					</div>
				{/if}

				{#if $generatedPrompt}
					<p><strong>Description:</strong> {$generatedPrompt}</p>
				{/if}

				<div class="preview">
					{#if $thumbnailUrl}
						<img src={$thumbnailUrl} alt="Generated Thumbnail" class="thumbnail" />
					{/if}
					<div class="download-container">
						{#if $imageDownloadUrl}
							<button
								class="download-button"
								on:click={() => window.open($imageDownloadUrl, '_blank')}
							>
								Download Generated Image (PNG)
							</button>
						{/if}

						{#if $svgDownloadUrl}
							<button
								class="download-button"
								on:click={() => window.open($svgDownloadUrl, '_blank')}
							>
								Download SVG File
							</button>
						{/if}

						{#if $gcodeDownloadUrl}
							<button
								class="download-button"
								on:click={() => window.open($gcodeDownloadUrl, '_blank')}
							>
								Download G-Code File
							</button>
						{/if}

						{#if $zipDownloadUrl}
							<button
								class="download-button"
								on:click={() => window.open($zipDownloadUrl, '_blank')}
							>
								Download All Files as ZIP
							</button>
						{/if}
					</div>
				</div>
			</div>
		</main>
	</div>
</div>

<style>
	.preview {
		display: flex;
	}
	.light {
		--bg-color: #f0f0f0;
		--text-color: #333;
		--button-bg: #007bff;
		--button-text: #fff;
		--error-bg: #ffe6e6;
		--error-border: #ff4d4d;
	}

	.dark {
		--bg-color: #1e1e1e;
		--text-color: #f0f0f0;
		--button-bg: #4a90e2;
		--button-text: #fff;
		--error-bg: #3a1a1a;
		--error-border: #e74c3c;
	}

	.container {
		background-color: var(--bg-color);
		color: var(--text-color);
		padding: 2rem;
		border-radius: 8px;
		max-width: 700px;
		margin: 2rem auto;
		box-shadow: 0px 4px 10px rgba(0, 0, 0, 0.1);
	}

	header {
		margin-bottom: 1.5rem;
	}

	.header-content {
		display: flex;
		justify-content: space-between;
		align-items: center;
	}

	h1 {
		margin: 0;
	}

	.theme-toggle {
		background: none;
		border: none;
		cursor: pointer;
		color: var(--text-color);
		font-size: 1.2rem;
	}

	.input-section {
		display: flex;
		flex-direction: column;
		gap: 1rem;
	}

	.concept-input {
		width: 100%;
		padding: 0.5rem;
		font-size: 1rem;
		border: 2px solid var(--text-color);
		border-radius: 4px;
	}

	.generate-button {
		width: 100%;
		padding: 0.7rem;
		font-size: 1.1rem;
		background-color: var(--button-bg);
		color: var(--button-text);
		border: none;
		border-radius: 4px;
		cursor: pointer;
	}

	.generate-button:disabled {
		background-color: grey;
		cursor: not-allowed;
	}

	.loading {
		color: var(--text-color);
		margin-top: 10px;
		font-style: italic;
	}

	.response-container {
		margin-top: 1.5rem;
	}

	.thumbnail {
		max-width: 100%;
		height: auto;
		margin-top: 1rem;
		border-radius: 8px;
		border: 1px solid var(--text-color);
		margin-right: 1rem;
	}

	.download-container {
		display: flex;
		flex-direction: column;
		gap: 0.5rem;
		margin-top: 1rem;
	}

	.download-button {
		width: 100%;
		padding: 0.7rem;
		font-size: 1rem;
		background-color: var(--button-bg);
		color: var(--button-text);
		border: none;
		border-radius: 4px;
		cursor: pointer;
	}

	.download-button:hover {
		background-color: darken(var(--button-bg), 10%);
	}

	.error-details {
		margin-top: 1rem;
		border: 1px solid var(--error-border);
		padding: 1rem;
		background-color: var(--error-bg);
		border-radius: 8px;
	}
</style>
