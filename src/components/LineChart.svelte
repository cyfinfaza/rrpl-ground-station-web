<script>
	import { onMount } from "svelte";
	import { AxisScrollStrategies, lightningChart } from "@arction/lcjs";
	import { basicTheme } from "../lib/chartTheme";

	export let data = {};
	export let randomDataMode = false;
	export let numRandomSamplesPerFrame = 4;

	let lineSeries = {};

	let chartElem;
	let chart;

	let numRandomDataPoints = 0;

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
		if (randomDataMode) addRandomData();
		numRandomDataPoints = lineSeries?.["rand0"]?.getPointAmount() || 0;
		requestAnimationFrame(animFrame);
	}

	function addRandomData() {
		["rand0", "rand1", "rand2", "rand3", "rand4", "rand5"].forEach(
			(dataset) => {
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
						new Array(numRandomSamplesPerFrame).fill(0).map((_, i) => ({
							x: lineSeries[dataset].getPointAmount() + i,
							y: Math.random() * 10,
						}))
					);
			}
		);
	}

	onMount(() => {
		chart = lightningChart()
			.ChartXY({ container: chartElem, theme: basicTheme })
			.setTitle("Sensor data");
		chart.addLegendBox().add(chart);
		chart
			.getDefaultAxisX()
			.setScrollStrategy(AxisScrollStrategies.progressive)
			.setInterval({
				start: 0,
				end: 400, // 60 seconds as milliseconds
				stopAxisAfter: false,
			});
		animFrame();
		const seriesInterval = setInterval(() => {
			chart.addLegendBox().add(chart);
		}, 5000);
		// setInterval(() => {
		// 	if (randomDataMode) addRandomData();
		// }, 50);
		return () => {
			chart.dispose();
			clearInterval(seriesInterval);
		};
	});
</script>

<div class="chart" bind:this={chartElem} />
<!-- {#if randomDataMode} -->
<p style="position: absolute; top: 0; right: 0;">
	{numRandomDataPoints} points
</p>

<!-- {/if} -->

<style>
	.chart {
		width: 100%;
		height: 100%;
	}
</style>
