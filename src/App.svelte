<script>
	import { onMount } from "svelte";
	import LineChart from "./components/LineChart.svelte";
	import getUsbId from "./lib/getUsbId";
	import { settings } from "./lib/stores";
	import { decodeMinervaIIPacket } from "./lib/decode";
	import { Line } from "svelte-chartjs";
	// import Battery from './components/Battery.svelte'

	let serialPort = null;
	let usbDeviceInfo = null;
	
	$: if (serialPort) {
		const portInfo = serialPort.getInfo();
		// console.log(portInfo);
		usbDeviceInfo = getUsbId(portInfo.usbVendorId, portInfo.usbProductId);
	}

	let serialDataStream = [];
	let stopReadingPlz = false;

	let data = {};

	let numDataPoints = 0;
	let numDataPointsWhileConnected = 0;
	let batteryCharge = 0;
	let batteryAvgChange = 0; //last second
	let batteryDeltaArr = [];
	let timeWhenConnected = 1;
	let time = {"hours": "00", "minutes": "00", "seconds": "00", "milliseconds": "00"};
	let timeElapsed=0;
	let isFirst = true;
	const logValues = {
		kf_acceleration_mss: "acceleration",
		kf_velocity_ms: "velocity",
		kf_position_m: "position",
		barometer_hMSL_m: "barometer",
		acceleration_z_mss: "z_acceleration",
		main_voltage_v:"battery_charge",
		time:"time_us"
	};

	function calculateAverageChange(arr) {
		let difference = 0;
		difference = arr[arr.length - 1] - arr[0];
		return Math.round((difference / (0.1*arr.length)) * 1000) / 1000 ;
		
	}

	function convertTime(milliseconds) {
		let seconds = Math.floor((milliseconds / 1000) % 60);
		let minutes = Math.floor((milliseconds / (1000 * 60)) % 60);
		let hours = Math.floor((milliseconds / (1000 * 60 * 60)) % 24);
		return {
			hours: hours,
			minutes: minutes,
			seconds: seconds
		};	
	}

	function updateDataFromSerialStream() {
		let newData = data;
		if (serialDataStream?.length > 0) {
			while (serialDataStream?.length > 0) {
				const line = serialDataStream.shift();
				if(isFirst){
					isFirst=false;
					continue;
				}
				try {
					// console.log(line);
					const decoded = decodeMinervaIIPacket(new Uint8Array(line).buffer);

					// console.log(decoded);
					Object.keys(logValues).forEach((logValue) => {
						if (!data[logValue]) {
							data[logValue] = [];
						}
						data[logValue] = [...data[logValue], decoded[logValue]];
					});
					//pwewse put this as a compotnent 
					numDataPointsWhileConnected++;
					batteryCharge = Math.round(100*data.main_voltage_v[data.main_voltage_v.length-1])/ 100;
					batteryDeltaArr.push(data.main_voltage_v[data.main_voltage_v.length-1])
					if(batteryDeltaArr.length > 10) {
						batteryDeltaArr.shift();
					}
					batteryAvgChange = calculateAverageChange(batteryDeltaArr); 	
					timeElapsed=Date.now()-timeWhenConnected;
					time = convertTime(timeElapsed);

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
			// console.log(serialPort);
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
							// console.log("Read done");
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
			// console.log(event);
		});
		navigator.serial.addEventListener("disconnect", (event) => {
			// console.log(event);
			if (event.port === serialPort) {
				serialPort = null;
			}
		});
	}

	onMount(() => {
		setInterval(() => {
			// console.log(serialDataStream, numDataPoints, timeWhenConnected);
			//console.log(data.main_voltage_v);
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
		
		<p>Battery Charge: {batteryCharge}<p>
		<p>Battery Delta: {batteryAvgChange}</p>

		<!-- {console.log(data.main_voltage_v)} -->
		<p>
			Data points: {numDataPoints} <br />
			Capture Rate: {(
				numDataPointsWhileConnected /
				((Date.now() - timeWhenConnected) / 1000)
			).toPrecision(3)} Hz
		</p>
			<img src='src/RRPLLogo.png' class='RRPLimg'> 
		
		
	</div>
	<div class="graphs stopwatch-holder">
		<LineChart
			{data}
			showJust="kf_acceleration_mss"
			title="Acceleration"
			nameMap={logValues}
		/>
		<div class="stopwatch-holder">
			<p>{time.hours}:{time.minutes}:{time.seconds}.{time.milliseconds}</p>
		</div>
		<LineChart
			{data}
			showJust="kf_velocity_ms"
			title="Velocity"
			nameMap={logValues}
		/>
		<LineChart
			{data}
			showJust="kf_position_m"
			title="Position"
			nameMap={logValues}
		/>
		<LineChart
			{data}
			showJust="barometer_hMSL_m"
			title="Barometric Altitude"
			nameMap={logValues} 
		/>
		<LineChart
			{data}
			showJust="acceleration_z_mss"
			title="Z Acceleration"
			nameMap={logValues} 
		/>
		
	</div>
</main>

<style lang="scss">
	@import url('https://fonts.googleapis.com/css2?family=Orbitron:wght@400..900&display=swap');

	main {
		width: 100vw;
		height: 100vh;
		display: grid;
		grid-template-columns: 300px 1fr;
		padding: var(--spacing);
		gap: var(--spacing);
		box-sizing: border-box;
	}
	.graphs {
		display: grid;
		grid-template-columns: 1fr 1fr;
		grid-template-rows: 0.5fr 0.5fr 0.5fr;
		gap: var(--spacing);
		> :global(*) {
			width: calc(100% - 4px);
			height: calc(100% - 4px);
			background: var(--graphs);
			border: 2px solid var(--fg);
			border-radius: 12px;
			/* box-sizing: border-box !important; */
		}
	}
	.graphs .stopwatch-holder {
		padding:0px;
		margin:0px;
		display:flex;
		justify-content:center;
		align-items:center;
		font-size:2rem;
		font-family: "Orbitron";
	}
	.controls {
		display: flex;
		flex-direction: column;
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
	.RRPLimg{
		padding-top:25rem;
	}
</style>