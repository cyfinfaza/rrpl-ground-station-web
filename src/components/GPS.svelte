
<script>
    import * as L from "leaflet";

    import { onMount } from "svelte";

    let data = {};
    let map;
    let marker;
    let mapElem;

    function updatePosition() {
        let lat = data.latitude[data.latitude.length-1];
        let lng = data.longitude[data.longitude.length-1];
        let latlngPnt = map.project([lat, lng], map.getMaxZoom() - 1);
        marker.LatLng(latlngPnt);
    }
    
    onMount(() => {
        // map = L.map(mapElem);
        // L.geoJSON(usa, {
        //     // maxZoom: 1,
        //     attribution: '&copy; <a href="http://www.openstreetmap.org/copyright">OpenStreetMap</a>'
        // }).addTo(map);
        // marker = L.marker([0,0]).addTo(map)
        map = L.map(mapElem, {
            minZoom: 1,
            maxZoom: 3,
            center: [0, 0],
            zoom: 1,
            crs: L.CRS.Simple,
        });
        var w = 1500;
        var h = 1000;
        var url = 'src/engineering.png';
        var southWest = map.unproject([ 0, h], map.getMaxZoom()-1);
        var northEast = map.unproject([ w, 0], map.getMaxZoom()-1);
        var bounds = new L.LatLngBounds( southWest, northEast);
    //    var bounds = [[0, 0], [1000, 1000]];
        L.imageOverlay( url, bounds, {
            attribution: '&copy; <a href="http://www.openstreetmap.org/copyright">OpenStreetMap</a>'

        }).addTo(map);
        

        map.setMaxBounds(bounds);
        let latlngPnt = map.project([40.52, -74.46]);
        let bnds = map.getBounds();
        
        marker = L.marker(map.unproject([500, 500], map.getMaxZoom() - 1)).addTo(map)
        //148.92, 81.04
        // map.fitBounds(bounds)
        // console.log(map.getBounds())

        updatePosition();
    })


</script>



<div class="map" bind:this={mapElem} style="width: 590px; height: 450px;">
</div>