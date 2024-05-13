
<script>
    import { usa } from "./usa";
    import * as L from "leaflet";

    import { onMount } from "svelte";

    let data = {};
    let map;
    let marker;
    let mapElem;

    function updatePosition() {
        let lat = data.latitude[data.latitude.length-1];
        let lng = data.longitude[data.longitude.length-1];
        let latlng = L.latLng(lat, lng);
        marker.LatLng(latlng);
    }
    
    onMount(() => {
        map = L.map(mapElem).setView([40.5, -74.5], 0.01);
        L.geoJSON(usa, {
            // maxZoom: 1,
            attribution: '&copy; <a href="http://www.openstreetmap.org/copyright">OpenStreetMap</a>'
        }).addTo(map);
        marker = L.marker([0,0]).addTo(map)
        updatePosition();
    })


</script>



<div class="map" bind:this={mapElem} style="width: 778px; height: 617px;">
</div>