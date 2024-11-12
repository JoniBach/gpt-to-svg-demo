<script>
	import { env } from '$env/dynamic/public';
	import { onMount } from 'svelte';

	const apiKey = env.PUBLIC_API_KEY;
	const apiUrl = env.PUBLIC_API_URL;
	const pingUrl = `${apiUrl}`.replace('generate', 'ping');  // Set the ping URL

	let responseMessage = '';
	let generatedPrompt = '';
	let thumbnailUrl = '';
	let imageDownloadUrl = '';
	let svgDownloadUrl = '';
	let gcodeDownloadUrl = '';
	let zipDownloadUrl = '';
	let progressMessage = '';
	let isLoading = false;
	let errorDetails = '';
	let startTime = '';
	let finishTime = '';

	let conceptInput = 'Default concept';
	let isDarkMode = false;
	let serverStarting = true;  // Indicates if the server is starting up
	let serverReady = false;  // Indicates if the server is ready

	if (typeof localStorage !== 'undefined') {
		const savedTheme = localStorage.getItem('theme');
		if (savedTheme === 'dark') {
			isDarkMode = true;
		}
	}

	const toggleTheme = () => {
		isDarkMode = !isDarkMode;
		if (typeof localStorage !== 'undefined') {
			localStorage.setItem('theme', isDarkMode ? 'dark' : 'light');
		}
	};

	const pingServer = async () => {
		try {
			const response = await fetch(pingUrl);
			if (response.ok) {
				return true; // Server responded successfully
			}
		} catch (error) {
			console.error('Ping failed:', error);
		}
		return false; // Server did not respond successfully
	};

	const generateImage = async () => {
		isLoading = true;
		progressMessage = 'Generating image, please wait...';
		errorDetails = '';
		startTime = new Date().toLocaleTimeString();
		finishTime = '';

		try {
			progressMessage = 'Sending request to generate the image...';

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
				responseMessage = 'Error: ' + response.status + ' - ' + errorText;
				progressMessage = 'Failed to generate image. Please try again.';
				errorDetails = `Status Code: ${response.status}, Error Text: ${errorText}`;
				isLoading = false;
				startTime = '';
				return;
			}

			progressMessage = 'Processing the response...';
			const data = await response.json();

			progressMessage = 'Updating the UI with the generated image links...';
			responseMessage = data.message;
			generatedPrompt = data.prompt;
			thumbnailUrl = data.thumbnail;
			imageDownloadUrl = data.image_download_url;
			svgDownloadUrl = data.svg_download_url;
			gcodeDownloadUrl = data.gcode_download_url;
			zipDownloadUrl = data.zip_download_url;

			progressMessage = 'Image generation complete!';
			finishTime = new Date().toLocaleTimeString();
			isLoading = false;
		} catch (error) {
			console.error('Network Error:', error);
			responseMessage = 'A network error occurred while generating the image.';
			progressMessage = 'Network error occurred. Please check your connection.';
			errorDetails = `Network Error: ${error.message}`;
			isLoading = false;
			startTime = '';
		}
	};

	onMount(async () => {
		// Immediately check server status on mount
		let success = await pingServer();
		if (success) {
			serverStarting = false;
			serverReady = true;
		} else {
			// Set up interval check every 10 seconds if server is not ready
			const checkInterval = setInterval(async () => {
				success = await pingServer();
				if (success) {
					clearInterval(checkInterval); // Stop checking when server is ready
					clearTimeout(stopCheckTimeout); // Stop the timeout as well
					serverStarting = false;
					serverReady = true;
				}
			}, 10000); // Ping every 10 seconds

			// Set a timeout to stop checking after 1 minute
			const stopCheckTimeout = setTimeout(() => {
				clearInterval(checkInterval); // Stop the interval after 1 minute
				serverStarting = false; // Hide the "Starting server" message
				progressMessage = 'Unable to reach the server. Please try again later.';
			}, 60000); // 1 minute timeout
		}
	});
</script>

<body class="{!isDarkMode ? 'dark' : 'light'} bg">
	<div class="container">
		<header>
			<div class="header-content">
				<h1>Image Generation API</h1>
				<button class="theme-toggle" on:click={toggleTheme}>
					{!isDarkMode ? '🌞 Light Mode' : '🌙 Dark Mode'}
				</button>
			</div>
		</header>

		<main>
			<!-- Display startup or ready message -->
			{#if serverStarting}
				<p class="server-message">Starting server, please wait...</p>
			{:else if serverReady}
				<p class="server-message">Server is ready to go!</p>
			{/if}

			<div class="input-section">
				<input
					type="text"
					placeholder="Enter your concept..."
					bind:value={conceptInput}
					class="concept-input"
					disabled={isLoading || serverStarting}
				/>
				<button on:click={generateImage} class="generate-button" disabled={isLoading || serverStarting}>
					{isLoading ? 'Generating...' : 'Generate Image'}
				</button>
			</div>

			{#if isLoading}
				<p class="loading">This could take a minute...</p>
			{/if}

			<div class="response-container">
				{#if responseMessage}
					<h2>Response</h2>
					<p>
						{responseMessage}
						{#if startTime && finishTime}
							in {Math.round(
								(new Date(`1970-01-01T${finishTime}Z`) - new Date(`1970-01-01T${startTime}Z`)) /
									1000
							)} seconds
						{/if}
					</p>
				{/if}

				{#if errorDetails}
					<div class="error-details">
						<p><strong>Error Details:</strong> {errorDetails}</p>
					</div>
				{/if}

				{#if generatedPrompt}
					<p><strong>Description:</strong> {generatedPrompt}</p>
				{/if}

				<div class="preview">
					{#if thumbnailUrl}
						<img src={thumbnailUrl} alt="Generated Thumbnail" class="thumbnail" />
					{/if}
					<div class="download-container">
						{#if imageDownloadUrl}
							<button
								class="download-button"
								on:click={() => window.open(imageDownloadUrl, '_blank')}
							>
								Download Generated Image (PNG)
							</button>
						{/if}

						{#if svgDownloadUrl}
							<button
								class="download-button"
								on:click={() => window.open(svgDownloadUrl, '_blank')}
							>
								Download SVG File
							</button>
						{/if}

						{#if gcodeDownloadUrl}
							<button
								class="download-button"
								on:click={() => window.open(gcodeDownloadUrl, '_blank')}
							>
								Download G-Code File
							</button>
						{/if}

						{#if zipDownloadUrl}
							<button
								class="download-button"
								on:click={() => window.open(zipDownloadUrl, '_blank')}
							>
								Download All Files as ZIP
							</button>
						{/if}
					</div>
				</div>
			</div>
		</main>
	</div>
</body>


<style>
	@import url('https://fonts.googleapis.com/css2?family=Montserrat:wght@400;700&display=swap');
	/* Add styling for server message */
	.server-message {
		font-size: 1.2rem;
		margin: 10px 0;
		color: var(--text-color);
		text-align: center;
	}
	body {
		font-family: 'Montserrat', sans-serif;
		background-color: #333;
		background: linear-gradient(135deg, #272727 60%, #1e1e1e);
		background-repeat: no-repeat;
		background-attachment: fixed;
	}
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
	.bg {
		display: flex;
		flex-direction: column;
		justify-content: center;
		height: 100vh;
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
		width: auto;
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
		width: 100%;
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
