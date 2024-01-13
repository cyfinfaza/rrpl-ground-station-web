<script>
	import { onMount } from "svelte";
	import LineChart from "./components/LineChart.svelte";
	import getUsbId from "./lib/getUsbId";
	import { settings } from "./lib/stores";
	import { decodeMinervaIIPacket } from "./lib/decode";

	let serialPort = null;
	let usbDeviceInfo = null;
	let randomDataMode = false;
	let numRandomSamplesPerFrame = 4;

	$: if (serialPort) {
		const portInfo = serialPort.getInfo();
		console.log(portInfo);
		usbDeviceInfo = getUsbId(portInfo.usbVendorId, portInfo.usbProductId);
	}

	let serialDataStream = [];
	let stopReadingPlz = false;

	let data = {};

	let numDataPoints = 0;
	let numDataPointsWhileConnected = 0;
	let timeWhenConnected = 1;

	const logValues = {
		kf_acceleration_mss: "acceleration",
		kf_velocity_ms: "velocity",
		kf_position_m: "position",
	};

	function updateDataFromSerialStream() {
		let newData = data;
		if (serialDataStream?.length > 0) {
			while (serialDataStream?.length > 0) {
				const line = serialDataStream.shift();
				try {
					// console.log(line);
					const decoded = decodeMinervaIIPacket(new Uint8Array(line).buffer);
					console.log(decoded);
					Object.keys(logValues).forEach((logValue) => {
						if (!data[logValue]) {
							data[logValue] = [];
						}
						data[logValue] = [...data[logValue], decoded[logValue]];
					});
					numDataPointsWhileConnected++;
				} catch (error) {
					console.error("Serial parse error", error);
				}
			}
		}
		numDataPoints = Object.values(newData)[0]?.length;
		// data = data.concat(newData);
		if (serialPort) requestAnimationFrame(updateDataFromSerialStream);
	}

	function initSerial() {
		navigator.serial.requestPort().then(async (port) => {
			serialPort = port;
			updateDataFromSerialStream();
			window.sp = port;
			console.log(serialPort);
			await port.open({ baudRate: parseInt($settings?.baudRate) || 115200 });
			let buffer = [];
			timeWhenConnected = Date.now();
			numDataPointsWhileConnected = 0;
			while (port.readable) {
				const reader = port.readable.getReader();
				try {
					stopReadingPlz = false;
					while (!stopReadingPlz) {
						const { value, done } = await reader.read();
						if (done) {
							console.log("Read done");
							break;
						}
						// console.log(value);
						let toAdd = buffer;
						// let string = new TextDecoder("utf-8").decode(value.slice(0, 2));
						// console.log(string, value.slice(0, 2));
						for (let i = 0; i < value.length; i++) {
							// console.log(toAdd.length, toAdd.slice(-1), toAdd.slice(-2));
							if (
								toAdd.length >= 107 &&
								toAdd.slice(-1) == 0xbe &&
								toAdd.slice(-2, -1) == 0xef
							) {
								toAdd.unshift(toAdd.pop());
								toAdd.unshift(toAdd.pop());
								serialDataStream.push(toAdd);
								// console.log(toAdd);
								toAdd = [];
							}
							toAdd.push(value[i]);
						}
						buffer = toAdd;
					}
				} catch (error) {
					console.error(error);
				} finally {
					reader.releaseLock();
					serialPort.close();
					serialPort = null;
					console.log("Reader released");
				}
			}
		});
		navigator.serial.addEventListener("connect", (event) => {
			console.log(event);
		});
		navigator.serial.addEventListener("disconnect", (event) => {
			console.log(event);
			if (event.port === serialPort) {
				serialPort = null;
			}
		});
	}

	onMount(() => {
		// setInterval(() => {
		// 	Object.keys(data).forEach((d) => {
		// 		data[d] = [...data[d], Math.random() * 10];
		// 	});
		// }, 5);
		setInterval(() => {
			// console.log(serialDataStream, numDataPoints, timeWhenConnected);
		}, 1000);
	});
</script>

<main>
	<div class="controls">
		<button on:click={initSerial}>
			{#if serialPort}
				{#if usbDeviceInfo}
					<small>Connected to</small> <br />
					<span>{usbDeviceInfo.vendor}</span>
					<br /> <strong>{usbDeviceInfo.product}</strong>
				{:else}
					Connected
				{/if}
			{:else}
				Click to connect serial (requires Chromium-based browser)
			{/if}
		</button>
		{#if serialPort}
			<button
				on:click={() => {
					stopReadingPlz = true;
					serialPort.close();
				}}>Disconnect</button
			>
		{/if}
		<p>
			Baud rate: <input
				type="number"
				bind:value={$settings.baudRate}
				placeholder="Baud Rate (115200)"
				style="width: 100px;"
			/>
		</p>
		<p>
			Data points: {numDataPoints} <br />
			Capture Rate: {(
				numDataPointsWhileConnected /
				((Date.now() - timeWhenConnected) / 1000)
			).toPrecision(3)} Hz
		</p>
		<h3>Low-level random data generator</h3>
		<p>
			Random samples per frame: <input
				type="number"
				bind:value={numRandomSamplesPerFrame}
				style="width: 100px;"
			/>
		</p>
		<button on:click={() => (randomDataMode = true)}>Start LLRD mode</button>
		<button on:click={() => (randomDataMode = false)}>Stop LLRD mode</button>
	</div>
	<div class="graphs">
		<LineChart
			{data}
			{randomDataMode}
			{numRandomSamplesPerFrame}
			nameMap={logValues}
		/>
		<!-- <LineChart {data} />
		<LineChart {data} />
		<LineChart {data} /> -->
	</div>
</main>

<style lang="scss">
	main {
		width: 100vw;
		height: 100vh;
		display: grid;
		grid-template-columns: 300px 1fr;
	}
	.graphs {
		display: grid;
		// grid-template-columns: 1fr 1fr;
		// grid-template-rows: 1fr 1fr;
	}

	.controls {
		display: flex;
		flex-direction: column;
		padding: 8px;
		gap: 8px;
		p,
		h1,
		h2,
		h3 {
			margin: 0;
		}
		h1,
		h2,
		h3 {
			margin-top: 4px;
		}
	}
</style>
