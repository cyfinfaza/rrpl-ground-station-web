<script>
  import { Line } from "svelte-chartjs";
  import {
    Chart as ChartJS,
    Title,
    Tooltip,
    Legend,
    LineElement,
    LinearScale,
    PointElement,
    CategoryScale,
  } from "chart.js";
  import { onMount } from "svelte";

  ChartJS.register(
    Title,
    Tooltip,
    Legend,
    LineElement,
    LinearScale,
    PointElement,
    CategoryScale
  );
  let data = {
    set1: [1, 2, 3, 4],
    set2: [2, 3, 4, 5],
    set3: [3, 4, 5, 6],
  };
  let chartData = {
    labels: [1, 2, 3, 4],
    datasets: [
      {
        label: "something",
        data: [2, 3, 4, 5, 5, 6],
      },
    ],
  };
  $: chartData = {
    labels: new Array(Object.values(data)[0].length)
      .fill(0)
      .map((_, i) => i)
      .slice(-500),
    datasets: Object.keys(data).map((d) => ({
      label: d,
      data: data[d].slice(-500),
    })),
  };
  // $: console.log(chartData);
  onMount(() => {
    setInterval(() => {
      Object.keys(data).forEach((d) => {
        data[d] = [...data[d], Math.random() * 10];
      });
    }, 5);
  });
</script>

<Line
  data={chartData}
  options={{
    animation: {
      duration: 0,
    },
  }}
/>
