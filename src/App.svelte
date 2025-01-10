<script>
	import { onMount } from "svelte";

	let ws;
	let localX = 0,
		localY = 0;
	let remoteX, remoteY;
	let action = "draw";
	let canvas, ctx;
	let isDrawing = false;
	const clientId = Math.random().toString(36).substr(2, 9);
	const distanceThreshold = 5;
	let reconnectAttempts = 10;
	const maxReconnectAttempts = 100;

	function initializeWebSocket() {
		ws = new WebSocket(
			"wss://web-production-efe2f.up.railway.app/ws/drawing/",
		);

		ws.onopen = () => console.log("WebSocket connected!");
		ws.onmessage = (event) => {
			const data = JSON.parse(event.data);

			if (data.clientId === clientId) {
				return;
			}

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

	// Simplified remote drawing handler
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
		ctx.strokeStyle = "black";
		ctx.lineWidth = 2;
		ctx.stroke();
	};

	const drawRemoteLine = (prevX, prevY, x, y) => {
		ctx.beginPath();
		ctx.moveTo(prevX, prevY);
		ctx.lineTo(x, y);
		ctx.strokeStyle = "blue";
		ctx.lineWidth = 2;
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
</script>

<meta
	name="viewport"
	content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no"
/>
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

<footer>
	<h2>About the Creator</h2>
	<p>
		This real-time drawing board is developed by <strong>Arin Gadre</strong
		>.
	</p>
	<ul>
		<li>
			<a href="https://www.linkedin.com/in/arin-gadre/" target="_blank"
				>LinkedIn</a
			>
		</li>
		<li>
			<a href="https://github.com/aringadre76" target="_blank">GitHub</a>
		</li>
		<li>
			<a href="mailto:aringadre@gmail.com">Email: aringadre@gmail.com</a>
		</li>
		<li>
			<a
				href="https://github.com/aringadre76/real-time-drawing-board-backend-deploy"
				target="_blank">Backend Code</a
			>
		</li>
		<li>
			<a
				href="https://github.com/aringadre76/real-time-drawing-board-frontend"
				target="_blank">Frontend Code</a
			>
		</li>
	</ul>
</footer>

<style>
	canvas {
		border: 1px solid #ccc;
		display: block;
		margin: 20px auto;
	}
	button {
		margin: 5px;
	}
	footer {
		margin-top: 30px;
		text-align: center;
	}
	footer ul {
		list-style: none;
		padding: 0;
	}
	footer li {
		margin: 5px 0;
	}
	footer a {
		text-decoration: none;
		color: #007acc;
	}
	footer a:hover {
		text-decoration: underline;
		color: #005f99;
	}
</style>
