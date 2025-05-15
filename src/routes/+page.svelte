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

	// Scale for circle radius (reactive)
	$: radiusScale = d3.scaleSqrt()
		.domain([0, d3.max(stations, d => d.totalTraffic) || 0])
		.range([0, 25]);

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
		try {
			const csvUrl = 'https://vis-society.github.io/labs/8/data/bluebikes-stations.csv';
			const data = await d3.csv(csvUrl);

			stations = data.map(station => ({
				id: station.Number,
				name: station.NAME,
				Lat: +station.Lat,
				Long: +station.Long,
			}));
		} catch (error) {
			console.error('Error loading station data:', error);
		}
	}

	async function loadStationDemand() {
		try {
			const csvUrl = 'https://vis-society.github.io/labs/8/data/bluebikes-traffic-2024-03.csv';
			const data = await d3.csv(csvUrl);

			trips = data.map(trip => ({
				id: trip.ride_id,
				started_at: new Date(trip.started_at),
				ended_at: new Date(trip.ended_at),
				start_station_id: trip.start_station_id,
				end_station_id: trip.end_station_id
			}));

			// Calcula partidas e chegadas
			departures = d3.rollup(trips, v => v.length, d => d.start_station_id);
			arrivals = d3.rollup(trips, v => v.length, d => d.end_station_id);

			// Atualiza estações com os dados de tráfego
			stations = stations.map(station => {
				const id = station.id;
				station.arrivals = arrivals.get(id) ?? 0;
				station.departures = departures.get(id) ?? 0;
				station.totalTraffic = station.arrivals + station.departures;
				return station;
			});
		} catch (error) {
			console.error('Error loading traffic data:', error);
		}
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

<h1>Welcome to SvelteKit</h1>
<p>Visit <a href="https://kit.svelte.dev">kit.svelte.dev</a> to read the documentation</p>

<div id="map">
	<svg>
		{#key mapViewChanged}
			{#each stations as station}
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
