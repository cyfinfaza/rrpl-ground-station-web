<script>
	import { onMount } from "svelte";
	import {
		AxisScrollStrategies,
		disableThemeEffects,
		lightningChart,
	} from "@arction/lcjs";
	import { basicTheme } from "../lib/chartTheme";

	export let data = {};
	export let randomDataMode = false;
	export let numRandomSamplesPerFrame = 4;
	export let nameMap = {};

	let lineSeries = {};

	let chartElem;
	let chart;
	let legendbox;

	let numRandomDataPoints = 0;

	function animFrame() {
		if (randomDataMode) addRandomData(numRandomSamplesPerFrame);
		else {
			Object.keys(data).forEach((dataset) => {
				if (!lineSeries[dataset]) {
					lineSeries[dataset] = chart
						.addLineSeries({
							// Optimize line series for progressively increasing X coordinates.
							dataPattern: { pattern: "ProgressiveX" },
						})
						.setName(nameMap[dataset] || dataset);
					legendbox.add(lineSeries[dataset]);
				}
				lineSeries[dataset]
					// .clear()
					.add(
						data[dataset]
							.slice(lineSeries[dataset].getPointAmount())
							.map((y, i) => ({
								x: i + lineSeries[dataset].getPointAmount(),
								y,
							})),
					);
			});
		}
		numRandomDataPoints = lineSeries?.["rand0"]?.getPointAmount() || 0;
		requestAnimationFrame(animFrame);
	}

	function addRandomData(num) {
		// ["rand0", "rand1", "rand2", "rand3", "rand4", "rand5"].forEach(
		// 	(dataset) => {
		// 		if (!lineSeries[dataset]) {
		// 			lineSeries[dataset] = chart
		// 				.addLineSeries({
		// 					// Optimize line series for progressively increasing X coordinates.
		// 					dataPattern: { pattern: "ProgressiveX" },
		// 				})
		// 				.setName(dataset);
		// 		}
		// 		lineSeries[dataset]
		// 			// .clear()
		// 			.add(
		// 				new Array(numRandomSamplesPerFrame).fill(0).map((_, i) => ({
		// 					x: lineSeries[dataset].getPointAmount() + i,
		// 					y: Math.random() * 10,
		// 				}))
		// 			);
		// 	}
		// );
		for (let randI = 0; randI < 6; randI++) {
			const dataset = `rand${randI}`;
			if (!lineSeries[dataset]) {
				console.log("Adding new line series", dataset);
				lineSeries[dataset] = chart
					.addLineSeries({
						// Optimize line series for progressively increasing X coordinates.
						dataPattern: { pattern: "ProgressiveX" },
					})
					.setName(dataset);
				legendbox.add(lineSeries[dataset]);
			}

			lineSeries[dataset].add(
				new Array(num).fill(0).map((_, i) => ({
					x: lineSeries[dataset].getPointAmount() + i,
					y: Math.random() * 10,
				})),
			);
		}
	}

	onMount(() => {
		chart = lightningChart()
			.ChartXY({ container: chartElem, theme: disableThemeEffects(basicTheme) })
			.setTitle("Sensor data");
		legendbox = chart.addLegendBox().add(chart);
		chart
			.getDefaultAxisX()
			.setScrollStrategy(AxisScrollStrategies.progressive)
			.setInterval({
				start: 0,
				end: 400, // 60 seconds as milliseconds
				stopAxisAfter: false,
			});
		// for (let i = 0; i < 24000; i++) {
		// 	addRandomData(50);
		// }
		animFrame();
		// const seriesInterval = setInterval(() => {
		// 	chart.addLegendBox().add(chart);
		// }, 5000);
		// setInterval(() => {
		// 	if (randomDataMode) addRandomData();
		// }, 15);
		return () => {
			chart.dispose();
			// clearInterval(seriesInterval);
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
