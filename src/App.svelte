<script>
	import { onMount } from "svelte";

	let ws;
	let localX = 0,
		localY = 0;
	let remoteX, remoteY;
	let action = "draw";
	let canvas, ctx;
	let isDrawing = false;
	let lineWidth = 2;
	let color = "#000000";
	const clientId = Math.random().toString(36).substr(2, 9);
	const distanceThreshold = 1;
	let reconnectAttempts = 10;
	const maxReconnectAttempts = 100;

	function initializeWebSocket() {
		ws = new WebSocket(
			"wss://real-time-drawing-board-backend-deploy.onrender.com/ws/drawing/",
		);

		ws.onopen = () => console.log("WebSocket connected!");
		ws.onmessage = (event) => {
			const data = JSON.parse(event.data);
			if (data.clientId === clientId) return;

			if (data.action === "draw") {
				handleRemoteDrawing(data.prevX, data.prevY, data.x, data.y);
			} else if (data.action === "erase") {
				eraseLine(data.x, data.y);
			}
		};

		ws.onerror = (error) => console.error("WebSocket error:", error);
		ws.onclose = () => {
			console.log("WebSocket disconnected.");
			reconnectWebSocket();
		};
	}

	function reconnectWebSocket() {
		if (reconnectAttempts < maxReconnectAttempts) {
			setTimeout(
				() => {
					initializeWebSocket();
					reconnectAttempts++;
				},
				1000 * Math.pow(2, reconnectAttempts),
			);
		}
	}

	const handleRemoteDrawing = (prevX, prevY, x, y) => {
		drawRemoteLine(prevX, prevY, x, y);
	};

	onMount(() => {
		canvas = document.querySelector("canvas");
		ctx = canvas.getContext("2d");
		ctx.lineJoin = "round";
		ctx.lineCap = "round";

		resizeCanvas();
		window.addEventListener("resize", resizeCanvas);
		initializeWebSocket();

		return () => {
			window.removeEventListener("resize", resizeCanvas);
			if (ws) ws.close();
		};
	});

	const startDrawing = (event) => {
		isDrawing = true;
		const rect = canvas.getBoundingClientRect();
		localX = event.clientX - rect.left;
		localY = event.clientY - rect.top;
	};

	const draw = (event) => {
		if (!isDrawing) return;

		const rect = canvas.getBoundingClientRect();
		const prevX = localX;
		const prevY = localY;
		localX = event.clientX - rect.left;
		localY = event.clientY - rect.top;

		if (action === "draw") {
			drawLocalLine(prevX, prevY, localX, localY);
		} else if (action === "erase") {
			eraseLine(localX, localY);
		}

		const distance = Math.hypot(localX - prevX, localY - prevY);

		if (distance >= distanceThreshold) {
			const payload = {
				prevX,
				prevY,
				x: localX,
				y: localY,
				action,
				clientId,
			};
			ws.send(JSON.stringify(payload));
		}
	};

	const stopDrawing = () => {
		isDrawing = false;
	};

	const drawLocalLine = (prevX, prevY, x, y) => {
		ctx.beginPath();
		ctx.moveTo(prevX, prevY);
		ctx.lineTo(x, y);
		ctx.strokeStyle = color;
		ctx.lineWidth = lineWidth;
		ctx.stroke();
	};

	const drawRemoteLine = (prevX, prevY, x, y) => {
		ctx.beginPath();
		ctx.moveTo(prevX, prevY);
		ctx.lineTo(x, y);
		ctx.strokeStyle = "#4A90E2";
		ctx.lineWidth = lineWidth;
		ctx.stroke();
	};

	const eraseLine = (x, y) => {
		ctx.save();
		ctx.beginPath();
		ctx.arc(x, y, 20, 0, Math.PI * 2);
		ctx.clip();
		ctx.clearRect(x - 20, y - 20, 40, 40);
		ctx.restore();
	};

	const resizeCanvas = () => {
		const container = canvas.parentElement;
		const dpr = window.devicePixelRatio || 1;
		canvas.width = container.clientWidth * dpr;
		canvas.height = window.innerHeight * 0.8 * dpr;
		canvas.style.width = `${container.clientWidth}px`;
		canvas.style.height = `${window.innerHeight * 0.8}px`;
		ctx.scale(dpr, dpr);
		ctx.lineJoin = "round";
		ctx.lineCap = "round";
	};

	const handleTouchStart = (event) => {
		event.preventDefault();
		const touch = event.touches[0];
		const rect = canvas.getBoundingClientRect();
		const scrollLeft =
			window.pageXOffset || document.documentElement.scrollLeft;
		const scrollTop =
			window.pageYOffset || document.documentElement.scrollTop;

		localX = touch.pageX - rect.left - scrollLeft;
		localY = touch.pageY - rect.top - scrollTop;
		isDrawing = true;
	};

	const handleTouchMove = (event) => {
		event.preventDefault();
		if (!isDrawing) return;

		const touch = event.touches[0];
		const rect = canvas.getBoundingClientRect();
		const scrollLeft =
			window.pageXOffset || document.documentElement.scrollLeft;
		const scrollTop =
			window.pageYOffset || document.documentElement.scrollTop;

		const prevX = localX;
		const prevY = localY;
		localX = touch.pageX - rect.left - scrollLeft;
		localY = touch.pageY - rect.top - scrollTop;

		if (action === "draw") {
			drawLocalLine(prevX, prevY, localX, localY);
		} else if (action === "erase") {
			eraseLine(localX, localY);
		}

		const distance = Math.hypot(localX - prevX, localY - prevY);

		if (distance > distanceThreshold) {
			const payload = {
				prevX,
				prevY,
				x: localX,
				y: localY,
				action,
				clientId,
			};
			ws.send(JSON.stringify(payload));
		}
	};

	const clearCanvas = () => {
		ctx.clearRect(0, 0, canvas.width, canvas.height);
	};
</script>

<meta
	name="viewport"
	content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no"
/>

<div class="container">
	<h1>Real-Time Drawing Board</h1>

	<div class="toolbar">
		<div class="tool-group">
			<button
				class="btn {action === 'draw' ? 'active' : ''}"
				on:click={() => (action = "draw")}
			>
				<svg
					xmlns="http://www.w3.org/2000/svg"
					width="24"
					height="24"
					viewBox="0 0 24 24"
					fill="none"
					stroke="currentColor"
					stroke-width="2"
				>
					<path
						d="M17 3a2.85 2.83 0 1 1 4 4L7.5 20.5 2 22l1.5-5.5L17 3z"
					/>
				</svg>
				Draw
			</button>
			<button
				class="btn {action === 'erase' ? 'active' : ''}"
				on:click={() => (action = "erase")}
			>
				<svg
					xmlns="http://www.w3.org/2000/svg"
					width="24"
					height="24"
					viewBox="0 0 24 24"
					fill="none"
					stroke="currentColor"
					stroke-width="2"
				>
					<path
						d="M20 20H7L3 16C2.5 15.5 2.5 14.5 3 14L13 4L20 11L11 20"
					/>
				</svg>
				Erase
			</button>
			<button class="btn" on:click={clearCanvas}>
				<svg
					xmlns="http://www.w3.org/2000/svg"
					width="24"
					height="24"
					viewBox="0 0 24 24"
					fill="none"
					stroke="currentColor"
					stroke-width="2"
				>
					<rect x="3" y="3" width="18" height="18" rx="2" ry="2" />
					<line x1="8" y1="12" x2="16" y2="12" />
				</svg>
				Clear
			</button>
		</div>

		<div class="tool-group">
			<label for="line-width">Line Width:</label>
			<input
				type="range"
				id="line-width"
				min="1"
				max="10"
				bind:value={lineWidth}
			/>

			<label for="color">Color:</label>
			<input type="color" id="color" bind:value={color} />
		</div>
	</div>

	<div class="canvas-container">
		<canvas
			width="800"
			height="600"
			on:mousedown={startDrawing}
			on:mousemove={draw}
			on:mouseup={stopDrawing}
			on:mouseleave={stopDrawing}
			on:touchstart|preventDefault={handleTouchStart}
			on:touchmove|preventDefault={handleTouchMove}
			on:touchend|preventDefault={stopDrawing}
		/>
	</div>

	<footer class="footer-container">
		<div class="creator-section">
			<h2>About the Creator</h2>
			<p>
				This real-time drawing board is developed by <strong
					>Arin Gadre</strong
				>
			</p>

			<div class="links-container">
				<a
					href="https://www.linkedin.com/in/arin-gadre/"
					target="_blank"
					class="profile-link"
				>
					<svg
						xmlns="http://www.w3.org/2000/svg"
						width="20"
						height="20"
						viewBox="0 0 24 24"
						fill="none"
						stroke="currentColor"
						stroke-width="2"
					>
						<path
							d="M16 8a6 6 0 0 1 6 6v7h-4v-7a2 2 0 0 0-2-2 2 2 0 0 0-2 2v7h-4v-7a6 6 0 0 1 6-6z"
						/>
						<rect x="2" y="9" width="4" height="12" />
						<circle cx="4" cy="4" r="2" />
					</svg>
					LinkedIn
				</a>
				<a
					href="https://github.com/aringadre76"
					target="_blank"
					class="profile-link"
				>
					<svg
						xmlns="http://www.w3.org/2000/svg"
						width="20"
						height="20"
						viewBox="0 0 24 24"
						fill="none"
						stroke="currentColor"
						stroke-width="2"
					>
						<path
							d="M9 19c-5 1.5-5-2.5-7-3m14 6v-3.87a3.37 3.37 0 0 0-.94-2.61c3.14-.35 6.44-1.54 6.44-7A5.44 5.44 0 0 0 20 4.77 5.07 5.07 0 0 0 19.91 1S18.73.65 16 2.48a13.38 13.38 0 0 0-7 0C6.27.65 5.09 1 5.09 1A5.07 5.07 0 0 0 5 4.77a5.44 5.44 0 0 0-1.5 3.78c0 5.42 3.3 6.61 6.44 7A3.37 3.37 0 0 0 9 18.13V22"
						/>
					</svg>
					GitHub
				</a>
				<a href="mailto:aringadre@gmail.com" class="profile-link">
					<svg
						xmlns="http://www.w3.org/2000/svg"
						width="20"
						height="20"
						viewBox="0 0 24 24"
						fill="none"
						stroke="currentColor"
						stroke-width="2"
					>
						<path
							d="M4 4h16c1.1 0 2 .9 2 2v12c0 1.1-.9 2-2 2H4c-1.1 0-2-.9-2-2V6c0-1.1.9-2 2-2z"
						/>
						<polyline points="22,6 12,13 2,6" />
					</svg>
					Email
				</a>
			</div>

			<div class="code-links">
				<a
					href="https://github.com/aringadre76/real-time-drawing-board-backend-deploy"
					target="_blank"
					class="code-link">Backend Code</a
				>
				<a
					href="https://github.com/aringadre76/real-time-drawing-board-frontend"
					target="_blank"
					class="code-link">Frontend Code</a
				>
			</div>
		</div>

		<div class="tech-stack">
			<h2>Technology Stack</h2>

			<div class="stack-grid">
				<div class="stack-card">
					<h3>Frontend</h3>
					<div class="stack-details">
						<div class="stack-item">
							<span class="label">Framework:</span>
							<span class="value">Svelte</span>
						</div>
						<div class="stack-item">
							<span class="label">Deployment:</span>
							<span class="value">Vercel</span>
						</div>
						<div class="stack-item">
							<span class="label">Features:</span>
							<span class="value"
								>Real-time WebSocket communication, Canvas API</span
							>
						</div>
					</div>
				</div>

				<div class="stack-card">
					<h3>Backend</h3>
					<div class="stack-details">
						<div class="stack-item">
							<span class="label">Framework:</span>
							<span class="value">Django Channels</span>
						</div>
						<div class="stack-item">
							<span class="label">Real-time Layer:</span>
							<span class="value">Redis</span>
						</div>
						<div class="stack-item">
							<span class="label">Deployment:</span>
							<span class="value">Render</span>
						</div>
						<div class="stack-item">
							<span class="label">Features:</span>
							<span class="value"
								>WebSocket handling, Real-time message
								broadcasting</span
							>
						</div>
					</div>
				</div>
			</div>
		</div>
	</footer>
</div>

<style>
	:global(body) {
		margin: 0;
		padding: 0;
		font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto,
			Oxygen, Ubuntu, Cantarell, "Open Sans", "Helvetica Neue", sans-serif;
		background-color: #f5f5f5;
		color: #333;
	}

	.container {
		max-width: 1200px;
		margin: 0 auto;
		padding: 1rem;
	}

	h1 {
		text-align: center;
		color: #2c3e50;
		margin-bottom: 1.5rem;
		font-size: 2rem;
	}

	.toolbar {
		background-color: white;
		padding: 1rem;
		border-radius: 8px;
		box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
		margin-bottom: 1rem;
		display: flex;
		flex-direction: column;
		gap: 1rem;
	}

	.tool-group {
		display: grid;
		gap: 0.5rem;
		width: 100%;
	}

	/* Main actions toolbar */
	.tool-group:first-child {
		grid-template-columns: repeat(3, 1fr);
	}

	/* Settings toolbar */
	.tool-group:nth-child(2) {
		display: grid;
		grid-template-columns: auto 1fr;
		gap: 0.5rem;
		align-items: center;
	}
	.btn {
		display: flex;
		align-items: center;
		justify-content: center;
		gap: 0.5rem;
		padding: 0.75rem;
		border: none;
		border-radius: 8px;
		background-color: #f0f0f0;
		color: #333;
		cursor: pointer;
		transition: all 0.2s ease;
		font-size: 0.9rem;
		white-space: nowrap;
	}

	.btn svg {
		width: 18px;
		height: 18px;
	}

	.btn:hover {
		background-color: #e0e0e0;
	}

	.btn.active {
		background-color: #4a90e2;
		color: white;
	}
	input[type="range"] {
		width: 100%;
		margin: 0;
	}
	label {
		font-size: 0.9rem;
		color: #666;
		white-space: nowrap;
	}

	@media (max-width: 480px) {
		.container {
			padding: 0.5rem;
		}

		h1 {
			font-size: 1.5rem;
			margin-bottom: 1rem;
		}

		.toolbar {
			padding: 0.75rem;
		}

		.btn {
			padding: 0.5rem;
			font-size: 0.8rem;
		}

		.btn svg {
			width: 16px;
			height: 16px;
		}

		/* Stack color and line width controls on very small screens */
		@media (max-width: 360px) {
			.tool-group:nth-child(2) {
				grid-template-columns: auto 1fr;
				row-gap: 0.75rem;
			}
		}
	}

	.canvas-container {
		background-color: white;
		border-radius: 8px;
		box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
		overflow: hidden;
	}

	.tech-stack {
		margin-top: 3rem;
		padding: 2rem;
		background-color: white;
		border-radius: 8px;
		box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
	}

	.tech-stack h3 {
		text-align: center;
		color: #2c3e50;
		margin-bottom: 1.5rem;
	}

	.footer-container {
		margin-top: 4rem;
		padding: 2rem;
		background-color: #f8f9fa;
		border-radius: 12px;
		box-shadow: 0 2px 15px rgba(0, 0, 0, 0.05);
	}

	.creator-section {
		text-align: center;
		margin-bottom: 3rem;
	}

	.creator-section h2 {
		color: #2d3748;
		font-size: 1.8rem;
		margin-bottom: 1rem;
	}

	.creator-section p {
		color: #4a5568;
		margin-bottom: 1.5rem;
	}

	.links-container {
		display: flex;
		justify-content: center;
		gap: 1.5rem;
		margin-bottom: 1.5rem;
	}

	.profile-link {
		display: flex;
		align-items: center;
		gap: 0.5rem;
		padding: 0.5rem 1rem;
		background-color: #fff;
		border: 1px solid #e2e8f0;
		border-radius: 6px;
		color: #4a5568;
		text-decoration: none;
		transition: all 0.2s ease;
	}

	.profile-link:hover {
		transform: translateY(-2px);
		box-shadow: 0 4px 12px rgba(0, 0, 0, 0.05);
		border-color: #4a90e2;
		color: #4a90e2;
	}

	.code-links {
		display: flex;
		justify-content: center;
		gap: 1rem;
	}

	.code-link {
		padding: 0.5rem 1rem;
		background-color: #4a90e2;
		color: white;
		text-decoration: none;
		border-radius: 6px;
		transition: all 0.2s ease;
	}

	.code-link:hover {
		background-color: #357abd;
		transform: translateY(-2px);
	}

	.tech-stack {
		margin-top: 3rem;
	}

	.tech-stack h2 {
		text-align: center;
		color: #2d3748;
		font-size: 1.8rem;
		margin-bottom: 2rem;
	}

	.stack-grid {
		display: grid;
		grid-template-columns: repeat(auto-fit, minmax(300px, 1fr));
		gap: 2rem;
		max-width: 1200px;
		margin: 0 auto;
	}

	.stack-card {
		background-color: white;
		border-radius: 8px;
		padding: 1.5rem;
		box-shadow: 0 2px 12px rgba(0, 0, 0, 0.05);
	}

	.stack-card h3 {
		color: #4a90e2;
		font-size: 1.4rem;
		margin-bottom: 1.5rem;
		padding-bottom: 0.5rem;
		border-bottom: 2px solid #f0f0f0;
	}

	.stack-details {
		display: flex;
		flex-direction: column;
		gap: 1rem;
	}

	.stack-item {
		display: flex;
		flex-direction: column;
		gap: 0.25rem;
	}

	.stack-item .label {
		color: #718096;
		font-size: 0.9rem;
		font-weight: 500;
	}

	.stack-item .value {
		color: #2d3748;
		font-size: 1rem;
	}

	@media (max-width: 768px) {
		.links-container {
			flex-direction: column;
			align-items: center;
		}

		.code-links {
			flex-direction: column;
			align-items: center;
		}

		.stack-grid {
			grid-template-columns: 1fr;
		}
	}
</style>
