<script>
	import { onMount } from "svelte";
	import { AxisScrollStrategies, lightningChart } from "@arction/lcjs";
	import { basicTheme } from "../lib/chartTheme";

	export let data = {};

	let lineSeries = {};

	let chartElem;
	let chart;

	function animFrame() {
		Object.keys(data).forEach((dataset) => {
			if (!lineSeries[dataset]) {
				lineSeries[dataset] = chart
					.addLineSeries({
						// Optimize line series for progressively increasing X coordinates.
						dataPattern: { pattern: "ProgressiveX" },
					})
					.setName(dataset);
			}
			lineSeries[dataset]
				// .clear()
				.add(
					data[dataset]
						.slice(lineSeries[dataset].getPointAmount())
						.map((y, i) => ({ x: i + lineSeries[dataset].getPointAmount(), y }))
				);
		});
		requestAnimationFrame(animFrame);
	}

	onMount(() => {
		chart = lightningChart()
			.ChartXY({ container: chartElem, theme: basicTheme })
			.setTitle("Svelte example");
		chart.addLegendBox().add(chart);
		chart
			.getDefaultAxisX()
			.setScrollStrategy(AxisScrollStrategies.progressive)
			.setInterval({
				start: 0,
				end: 200, // 60 seconds as milliseconds
				stopAxisAfter: false,
			});
		animFrame();
		return () => {
			chart.dispose();
		};
	});
</script>

<div class="chart" bind:this={chartElem} />

<style>
	.chart {
		width: 100%;
		height: 100%;
	}
</style>
