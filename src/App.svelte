<script>
	import { onMount } from "svelte";
	import LineChart from "./components/LineChart.svelte";
	import getUsbId from "./lib/getUsbId";
	import { settings } from "./lib/stores";

	let serialPort = null;
	let usbDeviceInfo = null;

	$: if (serialPort) {
		const portInfo = serialPort.getInfo();
		console.log(portInfo);
		usbDeviceInfo = getUsbId(portInfo.usbVendorId, portInfo.usbProductId);
	}

	let serialDataStream = [];
	let stopReadingPlz = false;

	let data = {
		// set1: [1, 2, 3, 4],
		// set2: [2, 3, 4, 5],
		// set3: [3, 4, 5, 6],
	};

	let numDataPoints = 0;
	let timeWhenConnected = 1;

	function updateDataFromSerialStream() {
		let newData = data;
		if (serialDataStream?.length > 0) {
			while (serialDataStream.length > 0) {
				const line = serialDataStream.shift();
				try {
					const sensorValues = JSON.parse(line);
					sensorValues.forEach((sensorValue, i) => {
						if (!newData[`sensor${i}`]) {
							newData[`sensor${i}`] = [];
						}
						newData[`sensor${i}`] = [...newData[`sensor${i}`], sensorValue];
					});
				} catch (error) {
					// console.error("Serial parse error", error);
				}
			}
		}
		numDataPoints = Object.values(newData)[0]?.length;
		// data = data.concat(newData);
		if (serialPort || randomDataInterval)
			requestAnimationFrame(updateDataFromSerialStream);
	}

	let randomDataInterval = null;
	function generateRandomData() {
		randomDataInterval = setInterval(() => {
			serialDataStream.push(
				JSON.stringify([
					Math.random() * 10,
					Math.random() * 10,
					Math.random() * 10,
					Math.random() * 10,
					Math.random() * 10,
					Math.random() * 10,
				])
			);
		}, 5);
		timeWhenConnected = Date.now();
		requestAnimationFrame(updateDataFromSerialStream);
	}

	function initSerial() {
		navigator.serial.requestPort().then(async (port) => {
			serialPort = port;
			updateDataFromSerialStream();
			window.sp = port;
			console.log(serialPort);
			await port.open({ baudRate: 115200 });
			let buffer = "";
			timeWhenConnected = Date.now();
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
						const rxString = new TextDecoder().decode(value);
						buffer += rxString;
						if (buffer.includes("\n")) {
							const lines = buffer.split("\n");
							serialDataStream.push(...lines.slice(0, lines.length - 1));
							buffer = lines[lines.length - 1];
						}
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
			console.log(serialDataStream, numDataPoints, timeWhenConnected);
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
				numDataPoints /
				((Date.now() - timeWhenConnected) / 1000)
			).toPrecision(3)} Hz
		</p>
		<button on:click={() => generateRandomData()}>Start random data</button>
		<button on:click={() => clearInterval(randomDataInterval)}
			>Stop random data</button
		>
	</div>
	<div class="graphs">
		<LineChart {data} />
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
		p {
			margin: 0;
		}
	}
</style>
