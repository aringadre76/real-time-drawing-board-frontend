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
	let drawQueue = []; // Queue for drawing operations
	let isAnimating = false; // Flag to track animation frame status
	let reconnectAttempts = 0;
	const maxReconnectAttempts = 5;
	
	// Initialize WebSocket connection
	function initializeWebSocket() {
		ws = new WebSocket("wss://web-production-efe2f.up.railway.app/ws/drawing/");
	
		ws.onopen = () => console.log("WebSocket connected!");
		ws.onmessage = (event) => {
			const data = JSON.parse(event.data);
	
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
	
		ws.onerror = (error) => {
			console.error("WebSocket error:", error);
			// Add user feedback about connection issues
		};
	
		ws.onclose = () => {
			console.log("WebSocket disconnected.");
			reconnectWebSocket();
		};
	}
	
	function reconnectWebSocket() {
		if (reconnectAttempts < maxReconnectAttempts) {
			setTimeout(() => {
				initializeWebSocket();
				reconnectAttempts++;
			}, 1000 * Math.pow(2, reconnectAttempts)); // Exponential backoff
		}
	}
	
	// Initialize everything on mount
	onMount(() => {
		// Canvas setup
		canvas = document.querySelector("canvas");
		ctx = canvas.getContext("2d");
		ctx.lineJoin = "round";
		ctx.lineCap = "round";
	
		// Canvas resize handling
		resizeCanvas();
		window.addEventListener("resize", resizeCanvas);
	
		// Initialize WebSocket
		initializeWebSocket();
	
		// Cleanup function
		return () => {
			window.removeEventListener("resize", resizeCanvas);
			if (ws) ws.close();
		};
	});
	
	// Process drawing queue using requestAnimationFrame
	const processDrawQueue = () => {
		while (drawQueue.length > 0) {
			const point = drawQueue.shift();
			if (point.isLocal) {
				drawLocalLine(point.prevX, point.prevY, point.x, point.y);
			} else {
				drawRemoteLine(point.prevX, point.prevY, point.x, point.y);
			}
		}
	
		if (drawQueue.length > 0) {
			requestAnimationFrame(processDrawQueue);
		} else {
			isAnimating = false;
		}
	};
	
	// Handle remote drawing
	const handleRemoteDrawing = (prevX, prevY, x, y) => {
		const startX = remoteX ?? prevX;
		const startY = remoteY ?? prevY;
	
		drawQueue.push({
			prevX: startX,
			prevY: startY,
			x,
			y,
			isLocal: false
		});
	
		remoteX = x;
		remoteY = y;
	
		if (!isAnimating) {
			isAnimating = true;
			requestAnimationFrame(processDrawQueue);
		}
	};
	
	// Start drawing locally
	const startDrawing = (event) => {
		isDrawing = true;
		const rect = canvas.getBoundingClientRect();
		localX = event.clientX - rect.left;
		localY = event.clientY - rect.top;
	};
	
	// Draw while moving locally
	const draw = (event) => {
		if (!isDrawing) return;
	
		const rect = canvas.getBoundingClientRect();
		const prevX = localX;
		const prevY = localY;
		localX = event.clientX - rect.left;
		localY = event.clientY - rect.top;
	
		// Add to draw queue
		drawQueue.push({
			prevX,
			prevY,
			x: localX,
			y: localY,
			isLocal: true
		});
	
		// Calculate the distance moved
		const distance = Math.hypot(localX - prevX, localY - prevY);
	
		// Send data only when distance exceeds the threshold
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
	
		// Start animation frame if not already running
		if (!isAnimating) {
			isAnimating = true;
			requestAnimationFrame(processDrawQueue);
		}
	};
	
	// Stop drawing locally
	const stopDrawing = () => {
		isDrawing = false;
	};
	
	// Draw a local line on the canvas
	const drawLocalLine = (prevX, prevY, x, y) => {
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
		ctx.clearRect(x - 5, y - 5, 10, 10);
	};
	
	// Initialize the canvas
	const resizeCanvas = () => {
		const container = canvas.parentElement;
		const dpr = window.devicePixelRatio || 1;
		canvas.width = container.clientWidth * dpr;
		canvas.height = (window.innerHeight * 0.8) * dpr;
		canvas.style.width = `${container.clientWidth}px`;
		canvas.style.height = `${window.innerHeight * 0.8}px`;
		ctx.scale(dpr, dpr); // Scale the context to ensure crisp rendering
	};
	
	// Touch event handlers with improved coordinate calculation
	const handleTouchStart = (event) => {
		event.preventDefault();
		const touch = event.touches[0];
		const rect = canvas.getBoundingClientRect();
		const scrollLeft = window.pageXOffset || document.documentElement.scrollLeft;
		const scrollTop = window.pageYOffset || document.documentElement.scrollTop;
	
		localX = (touch.pageX - rect.left - scrollLeft);
		localY = (touch.pageY - rect.top - scrollTop);
		isDrawing = true;
	};
	
	const handleTouchMove = (event) => {
		event.preventDefault();
		if (!isDrawing) return;
	
		const touch = event.touches[0];
		const rect = canvas.getBoundingClientRect();
		const scrollLeft = window.pageXOffset || document.documentElement.scrollLeft;
		const scrollTop = window.pageYOffset || document.documentElement.scrollTop;
	
		const prevX = localX;
		const prevY = localY;
		localX = (touch.pageX - rect.left - scrollLeft);
		localY = (touch.pageY - rect.top - scrollTop);
	
		// Add to draw queue
		drawQueue.push({
			prevX,
			prevY,
			x: localX,
			y: localY,
			isLocal: true
		});
	
		// Calculate distance and send data
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
	
		// Start animation frame if not already running
		if (!isAnimating) {
			isAnimating = true;
			requestAnimationFrame(processDrawQueue);
		}
	};
	</script>
	
	<meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no" />
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

	