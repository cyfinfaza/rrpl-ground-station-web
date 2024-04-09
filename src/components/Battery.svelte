<script>
	import { onMount } from "svelte";

	export let data = {"main_voltage_v": []};
    let toDisplay=8;
    let tag="main_voltage_v"
    //console.log(Object.keys(data).length);
    
    // toDisplay=data.main_voltage_v[data.main_voltage_v.length-1]

    //console.log(data.main_voltage_v[data.main_voltage_v.length-1]);

    onMount(() => {
        toDisplay = data.main_voltage_v[data.main_voltage_v.length-1];
    })
    

</script>
<p>Battery charge: {toDisplay}</p>