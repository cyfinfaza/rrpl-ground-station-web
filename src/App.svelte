<script>
	import { onMount } from "svelte";
	import LineChart from "./components/LineChart.svelte";

	let data = {
		set1: [1, 2, 3, 4],
		set2: [2, 3, 4, 5],
		set3: [3, 4, 5, 6],
	};

	onMount(() => {
		setInterval(() => {
			Object.keys(data).forEach((d) => {
				data[d] = [...data[d], Math.random() * 10];
			});
		}, 5);
	});
</script>

<main>
	<div class="controls">
		<button>Connect Serial Port<br />Hello</button>
	</div>
	<div class="graphs">
		<LineChart {data} />
		<LineChart {data} />
		<LineChart {data} />
		<LineChart {data} />
	</div>
</main>

<style>
	main {
		width: 100vw;
		height: 100vh;
		display: grid;
		grid-template-columns: 300px 1fr;
	}
	.graphs {
		display: grid;
		grid-template-columns: 1fr 1fr;
		grid-template-rows: 1fr 1fr;
	}

	.controls {
		display: flex;
		flex-direction: column;
		padding: 8px;
		gap: 8px;
	}
</style>
