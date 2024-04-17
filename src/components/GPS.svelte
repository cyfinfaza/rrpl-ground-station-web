
<script>
    import * as L from "leaflet";

    import { onMount } from "svelte";

    let map;
    let mapElem;
    
    onMount(() => {
        map = L.map(mapElem).setView([51.505, -0.09], 13);
        L.tileLayer('https://tile.openstreetmap.org/{z}/{x}/{y}.png', {
            maxZoom: 19,
            attribution: '&copy; <a href="http://www.openstreetmap.org/copyright">OpenStreetMap</a>'
        }).addTo(map);
    })


</script>



<div class="map" bind:this={mapElem}>
</div>