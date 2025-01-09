<script>
    import { onMount } from "svelte";

    let ws; // WebSocket connection
    let drawings = []; // Stores all received drawing data
    let x = 0, y = 0; // Coordinates for drawing actions
    let action = "draw"; // Action type (e.g., "draw" or "erase")
    let canvas, ctx; // Canvas and its context for rendering
    let isDrawing = false; // Tracks if the user is currently drawing
	let lastX, lastY; // Track the last point for WebSocket updates
	let lastSendTime = 0; // Track the last time a message was sent


    // Initialize WebSocket connection
    onMount(() => {
        ws = new WebSocket("wss://web-production-efe2f.up.railway.app/ws/drawing/");

        ws.onopen = () => console.log("WebSocket connected!");
		
		ws.onmessage = (event) => {
			const data = JSON.parse(event.data);

			if (data.action === "draw") {
				// Use the previous point or initialize if not set
				const prevX = lastX !== undefined ? lastX : data.x;
				const prevY = lastY !== undefined ? lastY : data.y;

				// Draw the line
				drawSmoothLine(prevX, prevY, data.x, data.y);

				// Update the last point
				lastX = data.x;
				lastY = data.y;
			} else if (data.action === "erase") {
				eraseLine(data.x, data.y);
			}
		};
        ws.onerror = (error) => console.error("WebSocket error:", error);
        ws.onclose = () => console.log("WebSocket disconnected.");
    });

    // Start drawing
    const startDrawing = (event) => {
        isDrawing = true;
        const rect = canvas.getBoundingClientRect();
        x = event.clientX - rect.left;
        y = event.clientY - rect.top;
    };

    // Draw while moving
	const draw = (event) => {
		if (!isDrawing) return;

		const rect = canvas.getBoundingClientRect();
		const prevX = x;
		const prevY = y;
		x = event.clientX - rect.left;
		y = event.clientY - rect.top;

		// Throttle the updates (send only every 50ms)
		const now = Date.now();
		const distance = Math.hypot(x - prevX, y - prevY); // Check movement distance

		if (now - lastSendTime > 50 && distance > 5) {
			const payload = { prevX, prevY, x, y, action };
			ws.send(JSON.stringify(payload)); // Send only if significant change
			lastSendTime = now;
		}

		if (action === "draw") {
			drawSmoothLine(prevX, prevY, x, y); // Draw locally
		} else if (action === "erase") {
			eraseLine(x, y); // Erase locally
		}
	};

    // Stop drawing
    const stopDrawing = () => {
        isDrawing = false;
    };

    // Draw a smooth line on the canvas
	const drawSmoothLine = (prevX, prevY, x, y) => {
		console.log(`Drawing line: (${prevX}, ${prevY}) to (${x}, ${y})`);
		ctx.beginPath();
		ctx.moveTo(prevX, prevY);
		ctx.lineTo(x, y);
		ctx.strokeStyle = "black";
		ctx.lineWidth = 2;
		ctx.stroke();
		ctx.closePath();
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

