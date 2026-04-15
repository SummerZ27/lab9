<svelte:options runes={false} />

<script>
	import { onMount } from "svelte";
	import mapboxgl from "mapbox-gl";
	import "../../node_modules/mapbox-gl/dist/mapbox-gl.css";
	import * as d3 from "d3";

	import { PUBLIC_MAPBOX_TOKEN } from "$env/static/public";
	mapboxgl.accessToken = PUBLIC_MAPBOX_TOKEN;

	let map;
	let stations = [];
	let mapViewChanged = 0;
	let timeFilter = -1;
	let selectedStation = null;
	let isochrone = null;
	let arrivals = new Map();
	let departures = new Map();

	const departuresByMinute = Array.from({length: 1440}, () => []);
	const arrivalsByMinute = Array.from({length: 1440}, () => []);

	$: timeFilterLabel = new Date(0, 0, 0, 0, timeFilter)
		.toLocaleString("en", {timeStyle: "short"});

	function minutesSinceMidnight(date) {
		return date.getHours() * 60 + date.getMinutes();
	}

	function filterByMinute(tripsByMinute, minute) {
		let minMinute = (minute - 60 + 1440) % 1440;
		let maxMinute = (minute + 60) % 1440;
		if (minMinute > maxMinute) {
			return tripsByMinute.slice(minMinute).concat(tripsByMinute.slice(0, maxMinute)).flat();
		}
		return tripsByMinute.slice(minMinute, maxMinute).flat();
	}

	$: filteredDepartures = timeFilter === -1
		? departures
		: d3.rollup(filterByMinute(departuresByMinute, timeFilter), v => v.length, d => d.start_station_id);

	$: filteredArrivals = timeFilter === -1
		? arrivals
		: d3.rollup(filterByMinute(arrivalsByMinute, timeFilter), v => v.length, d => d.end_station_id);

	$: filteredStations = stations.map(station => {
		const id = station.Number;
		const arr = filteredArrivals?.get(id) ?? 0;
		const dep = filteredDepartures?.get(id) ?? 0;
		return {
			...station,
			arrivals: arr,
			departures: dep,
			totalTraffic: arr + dep
		};
	});

	$: radiusScale = d3.scaleSqrt()
		.domain([0, d3.max(stations, d => d.totalTraffic) || 0])
		.range(timeFilter === -1 ? [0, 25] : [3, 30]);

	let stationFlow = d3.scaleQuantize()
		.domain([0, 1])
		.range([0, 0.5, 1]);

	function getCoords(station) {
		let point = new mapboxgl.LngLat(+station.Long, +station.Lat);
		let {x, y} = map.project(point);
		return {cx: x, cy: y};
	}

	const urlBase = "https://api.mapbox.com/isochrone/v1/mapbox/";
	const profile = "cycling";
	const minutes = [5, 10, 15, 20];
	const contourColors = ["03045e", "0077b6", "00b4d8", "90e0ef"];

	async function getIso(lon, lat) {
		const base = `${urlBase}${profile}/${lon},${lat}`;
		const params = new URLSearchParams({
			contours_minutes: minutes.join(","),
			contours_colors: contourColors.join(","),
			polygons: "true",
			access_token: mapboxgl.accessToken
		});
		const query = await fetch(`${base}?${params.toString()}`);
		isochrone = await query.json();
	}

	function geoJSONPolygonToPath(feature) {
		const path = d3.path();
		const rings = feature.geometry.coordinates;
		for (const ring of rings) {
			for (let i = 0; i < ring.length; i++) {
				const [lng, lat] = ring[i];
				const {x, y} = map.project([lng, lat]);
				if (i === 0) path.moveTo(x, y);
				else path.lineTo(x, y);
			}
			path.closePath();
		}
		return path.toString();
	}

	$: if (selectedStation) {
		getIso(+selectedStation.Long, +selectedStation.Lat);
	} else {
		isochrone = null;
	}

	const bikeLaneStyle = {
		"line-color": "green",
		"line-width": 3,
		"line-opacity": 0.4,
	};

	async function initMap() {
		map = new mapboxgl.Map({
			container: "map",
			center: [-71.09415, 42.36027],
			zoom: 12,
			style: "mapbox://styles/mapbox/streets-v12",
		});
		await new Promise(resolve => map.on("load", resolve));

		map.on("move", () => mapViewChanged++);

		map.addSource("boston_route", {
			type: "geojson",
			data: "https://bostonopendata-boston.opendata.arcgis.com/datasets/boston::existing-bike-network-2022.geojson?outSR=%7B%22latestWkid%22%3A3857%2C%22wkid%22%3A102100%7D",
		});
		map.addLayer({
			id: "boston_bike_lanes",
			type: "line",
			source: "boston_route",
			paint: bikeLaneStyle,
		});

		map.addSource("cambridge_route", {
			type: "geojson",
			data: "https://raw.githubusercontent.com/cambridgegis/cambridgegis_data/main/Recreation/Bike_Facilities/RECREATION_BikeFacilities.geojson",
		});
		map.addLayer({
			id: "cambridge_bike_lanes",
			type: "line",
			source: "cambridge_route",
			paint: bikeLaneStyle,
		});
	}

	async function loadData() {
		stations = await d3.csv("https://vis-society.github.io/labs/9/data/bluebikes-stations.csv");

		let trips = await d3.csv(
			"https://vis-society.github.io/labs/9/data/bluebikes-traffic-2024-03.csv"
		).then(trips => {
			for (let trip of trips) {
				trip.started_at = new Date(trip.started_at);
				trip.ended_at = new Date(trip.ended_at);
				let startedMinutes = minutesSinceMidnight(trip.started_at);
				departuresByMinute[startedMinutes].push(trip);
				let endedMinutes = minutesSinceMidnight(trip.ended_at);
				arrivalsByMinute[endedMinutes].push(trip);
			}
			return trips;
		});

		departures = d3.rollup(trips, v => v.length, d => d.start_station_id);
		arrivals = d3.rollup(trips, v => v.length, d => d.end_station_id);

		stations = stations.map(station => {
			let id = station.Number;
			station.arrivals = arrivals.get(id) ?? 0;
			station.departures = departures.get(id) ?? 0;
			station.totalTraffic = station.arrivals + station.departures;
			return station;
		});
	}

	onMount(() => {
		initMap();
		loadData();
	});
</script>

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
			{#if isochrone}
				{#each isochrone.features as feature}
					<path
						d={geoJSONPolygonToPath(feature)}
						fill={feature.properties.fillColor}
						fill-opacity="0.2"
						stroke="#000"
						stroke-opacity="0.5"
						stroke-width="1"
					>
						<title>{feature.properties.contour} minutes of biking</title>
					</path>
				{/each}
			{/if}
			{#if map}
				{#each filteredStations as station}
					<circle
						{...getCoords(station)}
						r={radiusScale(station.totalTraffic)}
						style="--departure-ratio: {station.totalTraffic > 0 ? stationFlow(station.departures / station.totalTraffic) : 0.5}"
						class={station.Number === selectedStation?.Number ? "selected" : ""}
						on:mousedown={() => selectedStation = selectedStation?.Number !== station.Number ? station : null}
					>
						<title>{station.totalTraffic} trips ({station.departures} departures, {station.arrivals} arrivals)</title>
					</circle>
				{/each}
			{/if}
		{/key}
	</svg>
</div>

<div class="legend">
	<span class="legend-title">Legend:</span>
	<div style="--departure-ratio: 1"><span>More departures</span></div>
	<div style="--departure-ratio: 0.5"><span>Balanced</span></div>
	<div style="--departure-ratio: 0"><span>More arrivals</span></div>
</div>

<style>
	header {
		display: flex;
		gap: 1em;
		align-items: baseline;
	}

	header h1 {
		margin: 0;
		white-space: nowrap;
	}

	header label {
		margin-left: auto;
		white-space: nowrap;
	}

	header time,
	header em {
		display: block;
		text-align: center;
	}

	header em {
		color: #888;
		font-style: italic;
	}

	#map {
		flex: 1;
	}

	#map svg {
		position: absolute;
		z-index: 1;
		width: 100%;
		height: 100%;
		pointer-events: none;
	}

	#map svg circle {
		fill-opacity: 0.6;
		stroke: white;
		pointer-events: auto;
		cursor: pointer;
		transition: opacity 0.2s ease;
		--color-departures: steelblue;
		--color-arrivals: darkorange;
		--color: color-mix(
			in oklch,
			var(--color-departures) calc(100% * var(--departure-ratio)),
			var(--color-arrivals)
		);
		fill: var(--color);
	}

	#map svg:has(circle.selected) circle:not(.selected) {
		opacity: 0.3;
	}

	#map svg path {
		pointer-events: auto;
	}

	.legend {
		display: flex;
		gap: 1em;
		align-items: center;
		margin-block: 1em;
	}

	.legend-title {
		font-weight: bold;
	}

	.legend > div {
		display: flex;
		align-items: center;
		gap: 0.3em;
		--color-departures: steelblue;
		--color-arrivals: darkorange;
		--color: color-mix(
			in oklch,
			var(--color-departures) calc(100% * var(--departure-ratio)),
			var(--color-arrivals)
		);
	}

	.legend > div::before {
		content: "";
		display: inline-block;
		width: 1em;
		height: 1em;
		border-radius: 50%;
		background: var(--color);
	}
</style>
