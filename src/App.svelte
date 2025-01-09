<script>
	import { onMount } from "svelte";

	let ws; // WebSocket connection
	let x = 0,
		y = 0; // Local coordinates for drawing actions
	let action = "draw"; // Action type (e.g., "draw" or "erase")
	let canvas, ctx; // Canvas and its context for rendering
	let isDrawing = false; // Tracks if the user is currently drawing
	let accumulatedDistance = 0; // Accumulates distance drawn for throttling
	const sendThreshold = 10; // Send updates every 10 pixels
	const clientId = Math.random().toString(36).substr(2, 9); // Unique client identifier
	let lastRemoteX, lastRemoteY; // Track last points for remote drawing

	// Initialize WebSocket connection
	onMount(() => {
		ws = new WebSocket("wss://web-production-efe2f.up.railway.app/ws/drawing/");

		ws.onopen = () => console.log("WebSocket connected!");

		ws.onmessage = (event) => {
			const data = JSON.parse(event.data);

			// Ignore messages from the same client
			if (data.clientId === clientId) return;

			if (data.action === "draw") {
				// Handle remote drawing with interpolation
				if (lastRemoteX !== undefined && lastRemoteY !== undefined) {
					interpolateAndDraw(lastRemoteX, lastRemoteY, data.x, data.y, "black");
				} else {
					drawLine(data.x, data.y, data.x, data.y, "black"); // Start point
				}

				// Update lastRemoteX and lastRemoteY
				lastRemoteX = data.x;
				lastRemoteY = data.y;
			} else if (data.action === "erase") {
				eraseLine(data.x, data.y);
			}
		};

		ws.onerror = (error) => console.error("WebSocket error:", error);
		ws.onclose = () => console.log("WebSocket disconnected.");
	});

	// Start drawing locally
	const startDrawing = (event) => {
		isDrawing = true;
		const rect = canvas.getBoundingClientRect();
		x = event.clientX - rect.left;
		y = event.clientY - rect.top;
		accumulatedDistance = 0; // Reset accumulated distance
	};

	// Draw while moving locally
	const draw = (event) => {
		if (!isDrawing) return;

		const rect = canvas.getBoundingClientRect();
		const prevX = x;
		const prevY = y;
		x = event.clientX - rect.left;
		y = event.clientY - rect.top;

		// Calculate the distance moved
		const distance = Math.hypot(x - prevX, y - prevY);
		accumulatedDistance += distance;

		// Send data only when accumulated distance exceeds the threshold
		if (accumulatedDistance >= sendThreshold) {
			const payload = { prevX, prevY, x, y, action, clientId };
			ws.send(JSON.stringify(payload));
			accumulatedDistance = 0; // Reset after sending
		}

		// Draw the local line
		drawLine(prevX, prevY, x, y, "black");
	};

	// Stop drawing locally
	const stopDrawing = () => {
		isDrawing = false;
	};

	// Draw a line on the canvas
	const drawLine = (prevX, prevY, x, y, color) => {
		ctx.beginPath();
		ctx.moveTo(prevX, prevY);
		ctx.lineTo(x, y);
		ctx.strokeStyle = color;
		ctx.lineWidth = 2;
		ctx.stroke();
		ctx.closePath();
	};

	// Interpolate and draw between two points
	const interpolateAndDraw = (prevX, prevY, x, y, color) => {
		const distance = Math.hypot(x - prevX, y - prevY);
		const steps = Math.ceil(distance / 5); // Adjust step size for smoother lines
		const deltaX = (x - prevX) / steps;
		const deltaY = (y - prevY) / steps;

		for (let i = 0; i <= steps; i++) {
			const interpolatedX = prevX + deltaX * i;
			const interpolatedY = prevY + deltaY * i;
			const nextX = interpolatedX + deltaX;
			const nextY = interpolatedY + deltaY;
			drawLine(interpolatedX, interpolatedY, nextX, nextY, color);
		}
	};

	// Erase by drawing a "clear" rectangle
	const eraseLine = (x, y) => {
		ctx.clearRect(x - 5, y - 5, 10, 10); // Adjust erase size
	};

	// Initialize the canvas
	onMount(() => {
		canvas = document.querySelector("canvas");
		ctx = canvas.getContext("2d");
		ctx.lineJoin = "round";
		ctx.lineCap = "round";
	});
</script>

<h1>Real-Time Drawing Board</h1>

<div>
	<button on:click={() => (action = "draw")}>Draw</button>
	<button on:click={() => (action = "erase")}>Erase</button>
</div>

<canvas
	width="800"
	height="600"
	on:mousedown={startDrawing}
	on:mousemove={draw}
	on:mouseup={stopDrawing}
	on:mouseleave={stopDrawing}
></canvas>

<!-- Stop drawing if mouse leaves canvas -->

<style>
	canvas {
		border: 1px solid #ccc;
		display: block;
		margin: 20px auto;
	}
	button {
		margin: 5px;
	}
</style>
