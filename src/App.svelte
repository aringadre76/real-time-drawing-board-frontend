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
			"wss://web-production-efe2f.up.railway.app/ws/drawing/",
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

	<footer>
		<h2>About the Creator</h2>
		<p>
			This real-time drawing board is developed by <strong
				>Arin Gadre</strong
			>
		</p>
		<div class="social-links">
			<a
				href="https://www.linkedin.com/in/arin-gadre/"
				target="_blank"
				class="social-link"
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
				class="social-link"
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
						d="M9 19c-5 1.5-5-2.5-7-3m14 6v-3.87a3.37 3.37 0 0 0-.94-2.61c3.14-.35 6.44-1.54 6.44-7A5.44 5.44 0 0 0 20 4.77 5.07 5.07 0 0 0 19.91 1S18.73.65 16 2.48a13.38 13.38 0 0 0-7 0C6.27.65 5.09 1 5.09 1A5.07 5.07 0 0 0 5 4.77a5.44 5.44 0 0 0-1.5 3.78c0 5.42 3.3 6.61 6.44 7A3.37 3.37 0 0 0 9 18.13V22"
					/>
				</svg>
				GitHub
			</a>
			<a href="mailto:aringadre@gmail.com" class="social-link">
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
						d="M4 4h16c1.1 0 2 .9 2 2v12c0 1.1-.9 2-2 2H4c-1.1 0-2-.9-2-2V6c0-1.1.9-2 2-2z"
					/>
					<polyline points="22,6 12,13 2,6" />
				</svg>
				Email
			</a>
			<div class="code-links">
				<a
					href="https://github.com/aringadre76/real-time-drawing-board-backend-deploy"
					target="_blank">Backend Code</a
				>
				<a
					href="https://github.com/aringadre76/real-time-drawing-board-frontend"
					target="_blank">Frontend Code</a
				>
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
		padding: 2rem;
	}

	h1 {
		text-align: center;
		color: #2c3e50;
		margin-bottom: 2rem;
		font-size: 2.5rem;
	}

	.toolbar {
		background-color: white;
		padding: 1rem;
		border-radius: 8px;
		box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
		margin-bottom: 1rem;
		display: flex;
		justify-content: space-between;
		align-items: center;
		flex-wrap: wrap;
		gap: 1rem;
	}

	.tool-group {
		display: flex;
		align-items: center;
		gap: 1rem;
	}

	.btn {
		display: flex;
		align-items: center;
		gap: 0.5rem;
		padding: 0.5rem 1rem;
		border: none;
		border-radius: 4px;
		background-color: #f0f0f0;
		color: #333;
		cursor: pointer;
		transition: all 0.2s ease;
	}

	.btn:hover {
		background-color: #e0e0e0;
	}

	.btn.active {
		background-color: #4a90e2;
		color: white;
	}

	.canvas-container {
		background-color: white;
		border-radius: 8px;
		box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
		overflow: hidden;
	}
</style>
