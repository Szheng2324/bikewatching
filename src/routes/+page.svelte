
<style>
    @import url("$lib/global.css");
   
</style>


<header>
    <h1>
        Bikewatching
    </h1>
    <label for="time-filter">Filter by time:</label>
    <input type="range" id="time-filter" name="time-filter" min="-1" max="1440" bind:value={timeFilter}>
    <time id="time-display">{timeFilterLabel}</time>
    {#if timeFilter === -1}
        <em>(any time)</em>
    {/if}


</header>
<div id="map">
	<svg>
        {#each filteredStations as station}
            {#key mapViewChanged}
                
                    <circle { ...getCoords(station) } r={radiusScale(station.totalTraffic)} fill="steelblue" style="--departure-ratio: { stationFlow(station.departures / station.totalTraffic) }" />
                
            {/key}
        {/each}   
        
    </svg>
</div>
<div class="legend">
	<div style="--departure-ratio: 1">More departures</div>
	<div style="--departure-ratio: 0.5">Balanced</div>
	<div style="--departure-ratio: 0">More arrivals</div>
</div>

<script>
    import mapboxgl from "mapbox-gl";
    import "../../node_modules/mapbox-gl/dist/mapbox-gl.css";
    mapboxgl.accessToken = "pk.eyJ1Ijoic3poZW5nMjMyNCIsImEiOiJjbHVyMWJlcG4wMmZoMnBwZHp3aXd6Ymk2In0.mbsvxXWq5F4omRr-HXLJAw";
    import { onMount } from "svelte";
    import * as d3 from "d3";
    
    let timeFilter = -1;
    $: timeFilterLabel = timeFilter !== -1 ? new Date(0, 0, 0, 0, timeFilter).toLocaleString("en", {timeStyle: "short"}) : '';
    function getCoords (station) {
        let point = new mapboxgl.LngLat(+station.Long, +station.Lat);
        let {x, y} = map.project(point);
        return {cx: x, cy: y};
    }
    
    function minutesSinceMidnight (date) {
        return date.getHours() * 60 + date.getMinutes();
    }
    let mapViewChanged = 0;
    let map;
    let stations = [];
    let trips = [];
    let arrivals;
    let departures;
    $: map?.on("move", evt => mapViewChanged++);
    let stationFlow = d3.scaleQuantize()
	    .domain([0, 1])
	    .range([0, 0.5, 1]);
    
    onMount(async() => {
        map = new mapboxgl.Map({
            
            container:"map",
            center: [-71.09415,42.36027],
            zoom:12,
            minZoom: 8,
            maxZoom: 18,
            style: "mapbox://styles/mapbox/streets-v12",
        });
    
        await new Promise(resolve => map.on("load", resolve));
        map.addSource("boston_route", {
            type: "geojson",
             data: "https://bostonopendata-boston.opendata.arcgis.com/datasets/boston::existing-bike-network-2022.geojson?outSR=%7B%22latestWkid%22%3A3857%2C%22wkid%22%3A102100%7D",
         });
         map.addLayer({
                 id: "bostonlayer", // A name for our layer (up to you)
                 type: "line", // one of the supported layer types, e.g. line, circle, etc.
                 source: "boston_route", // The id we specified in `addSource()`
                 paint: {
                     "line-width": 3,
                     "line-color": "green",
                     "line-opacity": 0.4,
                    
                 },
         });
         map.addSource("cambridge_route", {
             type: "geojson",
             data: "https://raw.githubusercontent.com/cambridgegis/cambridgegis_data/main/Recreation/Bike_Facilities/RECREATION_BikeFacilities.geojson",
         });
         map.addLayer({
                 id: "cambridgelayer", // A name for our layer (up to you)
                 type: "line", // one of the supported layer types, e.g. line, circle, etc.
                source: "cambridge_route", // The id we specified in `addSource()`
                 paint: {
                     "line-width": 3,
                     "line-color": "green",
                     "line-opacity": 0.4,
                    
                 },
         });
    
    stations = await d3.csv("https://vis-society.github.io/labs/8/data/bluebikes-stations.csv");
    trips = await d3.csv('https://vis-society.github.io/labs/8/data/bluebikes-traffic-2024-03.csv').then(trips => {
    for (let trip of trips) {
        trip.started_at = new Date(trip.started_at);
        trip.ended_at = new Date(trip.ended_at);
    }
    return trips;
    });
    departures = d3.rollup(trips, v => v.length, d => d.start_station_id);
    arrivals = d3.rollup(trips, v => v.length, d => d.end_station_id);


    stations = stations.map(station => {
        const id = station.Number;
        station.arrivals = arrivals.get(id) ?? 0;
        station.departures = departures.get(id) ?? 0;
        station.totalTraffic = station.arrivals + station.departures;
        return station
    })
});


   
    $: radiusScale = d3.scaleSqrt()
        .domain([0, d3.max(stations, d => d.totalTraffic)])
        .range([0, 25]);
    

    
    $: filteredTrips = timeFilter === -1? trips : trips.filter(trip => {
        let startedMinutes = minutesSinceMidnight(trip.started_at);
        let endedMinutes = minutesSinceMidnight(trip.ended_at);
        return Math.abs(startedMinutes - timeFilter) <= 60
            || Math.abs(endedMinutes - timeFilter) <= 60;
    });
    
    $: filteredArrivals = d3.rollup(filteredTrips, v => v.length, d => d.start_station_id);
    $: filteredDepartures = d3.rollup(filteredTrips, v => v.length, d => d.end_station_id);
    $: filteredStations = stations.map(station => {
        station = {...station}; 
        let id = station.Number;
        station.arrivals = filteredArrivals.get(id) ?? 0;
        station.departures = filteredDepartures.get(id) ?? 0;
        station.totalTraffic = station.arrivals + station.departures;
        console.log(station);
        return station;
    });

        


    

</script>

