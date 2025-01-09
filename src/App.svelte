<script>
	import { onMount } from "svelte";

	let ws; // WebSocket connection
	let localX = 0,
		localY = 0; // Local coordinates for drawing
	let remoteX, remoteY; // Remote coordinates for drawing
	let action = "draw"; // Action type (e.g., "draw" or "erase")
	let canvas, ctx; // Canvas and its context for rendering
	let isDrawing = false; // Tracks if the user is currently drawing
	const clientId = Math.random().toString(36).substr(2, 9); // Unique client identifier
	const distanceThreshold = 10; // Distance threshold for sending updates (pixels)

	// Initialize WebSocket connection
	onMount(() => {
		ws = new WebSocket(
			"wss://web-production-efe2f.up.railway.app/ws/drawing/",
		);

		ws.onopen = () => console.log("WebSocket connected!");
		ws.onmessage = (event) => {
			const data = JSON.parse(event.data);

			console.log("Message received - Data clientId:", data.clientId);
			console.log("Local clientId:", clientId);
			console.log("Are they equal?:", data.clientId === clientId);

			if (data.clientId === clientId) {
				console.log("Should be ignoring this message from same client");
				return;
			}

			if (data.action === "draw") {
				handleRemoteDrawing(data.prevX, data.prevY, data.x, data.y);
			} else if (data.action === "erase") {
				eraseLine(data.x, data.y);
			}
		};

		ws.onerror = (error) => console.error("WebSocket error:", error);
		ws.onclose = () => console.log("WebSocket disconnected.");
	});

	// Handle remote drawing
	const handleRemoteDrawing = (prevX, prevY, x, y) => {
		console.log(`Remote drawing: (${prevX}, ${prevY}) to (${x}, ${y})`);

		// Use previous remote points or initialize
		const startX = remoteX ?? prevX;
		const startY = remoteY ?? prevY;

		// Draw the received line
		drawRemoteLine(startX, startY, x, y);

		// Update remote coordinates
		remoteX = x;
		remoteY = y;
	};

	// Start drawing locally
	const startDrawing = (event) => {
		isDrawing = true;
		const rect = canvas.getBoundingClientRect();
		localX = event.clientX - rect.left;
		localY = event.clientY - rect.top;
		console.log("Local drawing started:", localX, localY);
	};

	// Draw while moving locally
	const draw = (event) => {
		if (!isDrawing) return;

		const rect = canvas.getBoundingClientRect();
		const prevX = localX;
		const prevY = localY;
		localX = event.clientX - rect.left;
		localY = event.clientY - rect.top;

		// Draw locally first
		drawLocalLine(prevX, prevY, localX, localY); // <-- Draw locally first

		// Calculate the distance moved
		const distance = Math.hypot(localX - prevX, localY - prevY);

		// Then send data only when distance exceeds the threshold
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
			console.log("Sent WebSocket message:", payload);
		}
	};

	// Stop drawing locally
	const stopDrawing = () => {
		isDrawing = false;
		console.log("Local drawing stopped.");
	};

	// Draw a local line on the canvas
	const drawLocalLine = (prevX, prevY, x, y) => {
		console.log(`Local drawing: (${prevX}, ${prevY}) to (${x}, ${y})`);
		ctx.beginPath();
		ctx.moveTo(prevX, prevY);
		ctx.lineTo(x, y);
		ctx.strokeStyle = "black"; // Local lines are always black
		ctx.lineWidth = 2;
		ctx.stroke();
		ctx.closePath();
	};

	// Draw a remote line on the canvas
	const drawRemoteLine = (prevX, prevY, x, y) => {
		console.log(`Remote drawing: (${prevX}, ${prevY}) to (${x}, ${y})`);
		ctx.beginPath();
		ctx.moveTo(prevX, prevY);
		ctx.lineTo(x, y);
		ctx.strokeStyle = "blue"; // Remote lines are blue
		ctx.lineWidth = 2;
		ctx.stroke();
		ctx.closePath();
	};

	// Erase by drawing a "clear" rectangle
	const eraseLine = (x, y) => {
		ctx.clearRect(x - 5, y - 5, 10, 10); // Adjust erase size
		console.log(`Erase at: (${x}, ${y})`);
	};

	// Initialize the canvas
	// Add this to your onMount function
	const resizeCanvas = () => {
		const container = canvas.parentElement;
		canvas.width = container.clientWidth;
		canvas.height = window.innerHeight * 0.8; // 80% of viewport height
	};

	onMount(() => {
		canvas = document.querySelector("canvas");
		ctx = canvas.getContext("2d");
		ctx.lineJoin = "round";
		ctx.lineCap = "round";

		// Add resize handling
		resizeCanvas();
		window.addEventListener("resize", resizeCanvas);

		// Clean up
		return () => {
			window.removeEventListener("resize", resizeCanvas);
		};
	});
	// Add these new touch handler functions:
	const handleTouchStart = (event) => {
		const touch = event.touches[0];
		const rect = canvas.getBoundingClientRect();
		localX = touch.clientX - rect.left;
		localY = touch.clientY - rect.top;
		isDrawing = true;
		console.log("Touch drawing started:", localX, localY);
	};

	const handleTouchMove = (event) => {
		if (!isDrawing) return;

		const touch = event.touches[0];
		const rect = canvas.getBoundingClientRect();
		const prevX = localX;
		const prevY = localY;
		localX = touch.clientX - rect.left;
		localY = touch.clientY - rect.top;

		// Calculate distance and send data, similar to mouse draw function
		const distance = Math.hypot(localX - prevX, localY - prevY);

		drawLocalLine(prevX, prevY, localX, localY);

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
	on:touchstart|preventDefault={handleTouchStart}
	on:touchmove|preventDefault={handleTouchMove}
	on:touchend|preventDefault={stopDrawing}
/>

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
