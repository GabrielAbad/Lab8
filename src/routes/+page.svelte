<script>
	import mapboxgl from "mapbox-gl";
	import * as d3 from "d3";
	import "../../node_modules/mapbox-gl/dist/mapbox-gl.css";
	import { onMount } from "svelte";

	mapboxgl.accessToken = "pk.eyJ1IjoiZ2FicmllbGFiYWQiLCJhIjoiY21hcGhlejRhMGlqZTJrb2Z3cmU3N29lcyJ9.BbCf0f6x2oOH6kWL8h3pRQ";

	let map;
	let stations = [];
	let trips = [];

	let arrivals, departures;
	let mapViewChanged = 0;

	let timeFilter = -1;
	$: timeFilterLabel = new Date(0, 0, 0, 0, timeFilter)
		.toLocaleString("en", { timeStyle: "short" });

	function minutesSinceMidnight(date) {
		return date.getHours() * 60 + date.getMinutes();
	}

	// Reactive filtered data
	$: filteredTrips = timeFilter === -1 ? trips : trips.filter(trip => {
		const started = minutesSinceMidnight(trip.started_at);
		const ended = minutesSinceMidnight(trip.ended_at);
		return Math.abs(started - timeFilter) <= 60 || Math.abs(ended - timeFilter) <= 60;
	});

	$: filteredDepartures = d3.rollup(filteredTrips, v => v.length, d => d.start_station_id);
	$: filteredArrivals = d3.rollup(filteredTrips, v => v.length, d => d.end_station_id);

	$: filteredStations = stations.map(s => {
		let station = { ...s }; // clone para não mutar os dados originais
		const id = station.id;
		station.arrivals = filteredArrivals.get(id) ?? 0;
		station.departures = filteredDepartures.get(id) ?? 0;
		station.totalTraffic = station.arrivals + station.departures;
		return station;
	});

	// Raio dos círculos, dependente do filtro
	$: radiusScale = d3.scaleSqrt()
		.domain([0, d3.max(filteredStations, d => d.totalTraffic) || 0])
		.range(timeFilter === -1 ? [0, 25] : [3, 30]);

	async function initMap() {
		map = new mapboxgl.Map({
			container: 'map',
			center: [-71.09415, 42.36027],
			zoom: 12,
			style: "mapbox://styles/mapbox/streets-v12",
		});

		await new Promise(resolve => map.on("load", resolve));

		map.addSource("boston_route", {
			type: "geojson",
			data: "https://bostonopendata-boston.opendata.arcgis.com/datasets/boston::existing-bike-network-2022.geojson?outSR=%7B%22latestWkid%22%3A3857%2C%22wkid%22%3A102100%7D",
		});

		map.addLayer({
			id: "bike_routes",
			type: "line",
			source: "boston_route",
			paint: {
				"line-color": "#ff7f00",
				"line-width": 2,
			},
		});

		map.on("move", () => {
			mapViewChanged++;
		});
	}

	async function loadStationData() {
		const csvUrl = 'https://vis-society.github.io/labs/8/data/bluebikes-stations.csv';
		const data = await d3.csv(csvUrl);

		stations = data.map(station => ({
			id: station.Number,
			name: station.NAME,
			Lat: +station.Lat,
			Long: +station.Long,
		}));
	}

	async function loadStationDemand() {
		const csvUrl = 'https://vis-society.github.io/labs/8/data/bluebikes-traffic-2024-03.csv';
		trips = await d3.csv(csvUrl).then(trips => {
			for (let trip of trips) {
				trip.started_at = new Date(trip.started_at);
				trip.ended_at = new Date(trip.ended_at);
			}
			return trips;
		});

		departures = d3.rollup(trips, v => v.length, d => d.start_station_id);
		arrivals = d3.rollup(trips, v => v.length, d => d.end_station_id);

		// Estações com dados de tráfego inicial
		stations = stations.map(station => {
			const id = station.id;
			station.arrivals = arrivals.get(id) ?? 0;
			station.departures = departures.get(id) ?? 0;
			station.totalTraffic = station.arrivals + station.departures;
			return station;
		});
	}

	function getCoords(station) {
		if (!map) return { cx: 0, cy: 0 };
		const point = new mapboxgl.LngLat(station.Long, station.Lat);
		const { x, y } = map.project(point);
		return { cx: x, cy: y };
	}

	onMount(async () => {
		await initMap();
		await loadStationData();
		await loadStationDemand();
	});
</script>

<style>
	@import url("$lib/global.css");

	header {
		display: flex;
		align-items: baseline;
		gap: 1em;
		margin-bottom: 1em;
	}

	label {
		margin-left: auto;
	}

	time, em {
		display: block;
	}

	em {
		color: #888;
		font-style: italic;
	}

	#map {
		position: relative;
		width: 100%;
		height: 600px;
	}

	#map svg {
		position: absolute;
		z-index: 1;
		width: 100%;
		height: 100%;
		pointer-events: none;
	}
</style>

<header>
	<h1>🚲 BikeWatch</h1>
	<label>
		Filter by time:
		<input type="range" min="-1" max="1440" bind:value={timeFilter} />
		{#if timeFilter !== -1}
			<time>{timeFilterLabel}</time>
		{:else}
			<em>(any time)</em>
		{/if}
	</label>
</header>

<div id="map">
	<svg>
		{#key mapViewChanged}
			{#each filteredStations as station}
				<circle
					{...getCoords(station)}
					r={radiusScale(station.totalTraffic)}
					fill="steelblue"
					stroke="white"
					stroke-width="1"
				/>
			{/each}
		{/key}
	</svg>
</div>

