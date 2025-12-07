<template>
  <div class="app-root">
    <header
      class="app-header"
      role="banner"
    >
      <h1>🚻 Public Toilet Finder 🗺️</h1>
    </header>

    <!-- Map Area -->
    <main
      id="map"
      class="map-container"
      aria-label="Map showing toilets"
      tabindex="0"
    />

    <!-- Popup for feature info -->
    <section
      id="popup"
      class="ol-popup"
      role="dialog"
      aria-live="polite"
    />

    <!-- Navigation Panel -->
    <nav
      v-if="instructions.length"
      id="nav-panel"
      class="nav-panel"
      aria-label="Navigation Instructions"
    >
      <button
        class="toggle-nav"
        @click="showNav = !showNav"
      >
        {{ showNav ? 'Hide' : 'Show' }} Navigation
      </button>
      <div v-show="showNav">
        <h3>Navigation</h3>
        <ol>
          <li
            v-for="(instruction, index) in instructions"
            :key="index"
          >
            {{ instruction }}
          </li>
        </ol>
      </div>
    </nav>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted } from 'vue';
import Map from 'ol/Map';
import View from 'ol/View';
import TileLayer from 'ol/layer/Tile';
import ImageLayer from 'ol/layer/Image';
import ImageWMS from 'ol/source/ImageWMS';
import OSM from 'ol/source/OSM';
import VectorSource from 'ol/source/Vector';
import VectorLayer from 'ol/layer/Vector';
import GeoJSON from 'ol/format/GeoJSON';
import Overlay from 'ol/Overlay';
import Feature from 'ol/Feature';
import { Point, LineString } from 'ol/geom';
import { Style, Icon, Circle, Fill, Stroke } from 'ol/style';
import { fromLonLat } from 'ol/proj';

type Maneuver = { instruction: string }

// ---------------- STATE ----------------
const instructions = ref<string[]>([]);
const showNav = ref(true);

// ---------------- ROUTE LAYER ----------------
const routeSource = new VectorSource();
const routeLayer = new VectorLayer({
	source: routeSource,
	style: new Style({
		stroke: new Stroke({
			color: '#0077ff',
			width: 4
		})
	})
});

// ---------------- POLYLINE DECODER ----------------
function decodePolyline(encoded: string, precision = 1e6): number[][] {
	const coordinates: number[][] = [];
	let index = 0;
	let lat = 0;
	let lng = 0;

	while (index < encoded.length) {
		let result = 0;
		let shift = 0;
		let b: number;
		do {
			b = encoded.charCodeAt(index++) - 63;
			result |= (b & 0x1f) << shift;
			shift += 5;
		} while (b >= 0x20);
		const deltaLat = (result & 1) ? ~(result >> 1) : (result >> 1);
		lat += deltaLat;

		result = 0;
		shift = 0;
		do {
			b = encoded.charCodeAt(index++) - 63;
			result |= (b & 0x1f) << shift;
			shift += 5;
		} while (b >= 0x20);
		const deltaLon = (result & 1) ? ~(result >> 1) : (result >> 1);
		lng += deltaLon;

		coordinates.push([lat / precision, lng / precision]);
	}
	return coordinates;
}

// ---------------- MOUNT MAP ----------------
onMounted(() => {
	const popupEl = document.getElementById('popup') as HTMLElement;
	const popupOverlay = new Overlay({
		element: popupEl,
		positioning: 'bottom-center',
		offset: [0, -10]
	});

	const toiletsSource = new VectorSource();
	const toiletsLayer = new VectorLayer({
		source: toiletsSource,
		style: new Style({
			image: new Icon({
				src: '/toilet.png',
				scale: 0.05,
				anchor: [0.5, 1]
			})
		})
	});

	const wmsLayer = new ImageLayer({
		source: new ImageWMS({
			url: 'https://ahocevar.com/geoserver/wms',
			params: { LAYERS: 'ne:ne_10m_admin_0_countries' },
			ratio: 1,
			serverType: 'geoserver'
		}),
		opacity: 0.6
	});

	const userLocationSource = new VectorSource();
	const userLocationLayer = new VectorLayer({
		source: userLocationSource,
		style: new Style({
			image: new Circle({
				radius: 8,
				fill: new Fill({ color: 'blue' }),
				stroke: new Stroke({ color: 'white', width: 2 })
			})
		})
	});

	const map = new Map({
		target: 'map',
		layers: [
			new TileLayer({ source: new OSM() }),
			wmsLayer,
			toiletsLayer,
			userLocationLayer,
			routeLayer
		],
		overlays: [popupOverlay],
		view: new View({
			center: fromLonLat([10, 51]),
			zoom: 6
		})
	});

	let userLat = 0;
	let userLon = 0;

	if ('geolocation' in navigator) {
		navigator.geolocation.getCurrentPosition(
			(pos) => {
				userLat = pos.coords.latitude;
				userLon = pos.coords.longitude;

				const userPoint = new Point(fromLonLat([userLon, userLat]));
				const userFeature = new Feature({ geometry: userPoint, type: 'user' });

				userLocationSource.clear();
				userLocationSource.addFeature(userFeature);

				map.getView().setCenter(fromLonLat([userLon, userLat]));
				map.getView().setZoom(14);
			},
			() => alert('Location permission denied!')
		);
	}

	void fetch(
		'http://localhost:8080/geoserver/germany/ows?service=WFS&version=1.0.0&request=GetFeature&typeName=germany:germany_clean_geom&outputFormat=application/json&maxFeatures=50000'
	)
		.then((r) => r.json())
		.then((json) => {
			const features = new GeoJSON({
				dataProjection: 'EPSG:4326',
				featureProjection: 'EPSG:3857'
			}).readFeatures(json);
			features.forEach((f) => f.set('type', 'toilet'));
			toiletsSource.addFeatures(features);
		})
		.catch(() => {
			/* optional fallback */
		});

	async function drawRoute(uLat: number, uLon: number, tLat: number, tLon: number) {
		routeSource.clear();

		const req = {
			locations: [
				{ lat: uLat, lon: uLon },
				{ lat: tLat, lon: tLon }
			],
			costing: 'pedestrian',
			directions_options: { units: 'kilometers' }
		};

		const valhallaUrl = 'https://valhalla1.openstreetmap.de/route?json=' + encodeURIComponent(JSON.stringify(req));

		try {
			const res = await fetch(valhallaUrl);
			if (!res.ok) throw new Error(`Routing failed (${res.status})`);
			const data = await res.json();

			const leg = data.trip.legs[0];
			const shape = leg.shape as string;
			const maneuvers = leg.maneuvers as Maneuver[];

			const coords = decodePolyline(shape).map(([lat, lon]) => fromLonLat([lon, lat]));
			const line = new LineString(coords);
			routeSource.addFeature(new Feature(line));

			map.getView().fit(line.getExtent(), { padding: [50, 50, 50, 50] });

			instructions.value = maneuvers.map((m) => m.instruction);
		} catch (err) {
			console.error('Route error:', err);
			alert('Failed to get route.');
		}
	}

	// Accessible singleclick handler
	map.on('singleclick', (evt) => {
		while (popupEl.firstChild) popupEl.removeChild(popupEl.firstChild);

		const clicked = map.getFeaturesAtPixel(evt.pixel);
		if (!clicked || !clicked[0]) {
			popupOverlay.setPosition(undefined);
			return;
		}

		const feature = clicked[0] as Feature;
		const propsRaw = feature.getProperties() as Record<string, unknown>;
		if (propsRaw.type !== 'toilet') {
			popupOverlay.setPosition(undefined);
			return;
		}

		const name = String(propsRaw['name'] ?? 'Public Toilet');
		const access = String(propsRaw['access'] ?? 'Unknown');
		const wheelchair = String(propsRaw['wheelchair'] ?? 'Unknown');
		const hours = String(propsRaw['opening_hours'] ?? 'Not provided');
		const fee = String(propsRaw['fee'] ?? 'Unknown');
		const toiletLat = Number(propsRaw['ycoord'] ?? 0);
		const toiletLon = Number(propsRaw['xcoord'] ?? 0);

		const title = document.createElement('h3');
		title.textContent = name;
		title.className = 'popup-title';
		title.setAttribute('tabindex', '-1');

		const details = document.createElement('div');
		details.className = 'popup-details';
		details.innerHTML = `
			<p><strong>Access:</strong> ${access}</p>
			<p><strong>Wheelchair:</strong> ${wheelchair}</p>
			<p><strong>Hours:</strong> ${hours}</p>
			<p><strong>Fee:</strong> ${fee}</p>
		`;

		const buttonWrap = document.createElement('div');
		buttonWrap.className = 'popup-actions';

		const navigateButton = document.createElement('button');
		navigateButton.type = 'button';
		navigateButton.id = 'navigate-btn';
		navigateButton.textContent = 'Navigate';
		navigateButton.className = 'btn-navigate';
		navigateButton.setAttribute('aria-label', `Navigate to ${name}`);

		buttonWrap.appendChild(navigateButton);

		popupEl.appendChild(title);
		popupEl.appendChild(details);
		popupEl.appendChild(buttonWrap);

		popupEl.setAttribute('role', 'dialog');
		popupEl.setAttribute('aria-live', 'polite');
		popupEl.setAttribute('aria-label', `Details for ${name}`);
		popupEl.tabIndex = -1;

		popupOverlay.setPosition(evt.coordinate);

		setTimeout(() => {
			navigateButton.focus();
		}, 50);

		const navigateHandler = () => {
			if (userLat === 0 && userLon === 0) {
				alert('User location still loading...');
				return;
			}
			drawRoute(userLat, userLon, toiletLat, toiletLon);
		};

		const freshButton = navigateButton.cloneNode(true) as HTMLButtonElement;
		navigateButton.parentNode?.replaceChild(freshButton, navigateButton);

		freshButton.addEventListener('click', navigateHandler);
		freshButton.addEventListener('keydown', (ev) => {
			if (ev.key === 'Enter' || ev.key === ' ') {
				ev.preventDefault();
				navigateHandler();
			}
		});
	});
});
</script>

<style>
/* Top-Centered Header */
.app-header {
	position: fixed;
	top: 10px;
	left: 50%;
	transform: translateX(-50%);
	background: #ADD8E6;
	padding: 12px 25px;
	border-radius: 12px;
	box-shadow: 0 4px 10px rgba(0,0,0,0.2);
	text-align: center;
	z-index: 10000;
}

.app-header h1 {
	margin: 0;
	font-size: 24px;
	color: #003366;
}

@media (max-width: 768px) {
	.app-header {
		padding: 10px 15px;
	}
	.app-header h1 {
		font-size: 20px;
	}
}

/* Map Container */
.map-container {
	width: 100%;
	height: 100vh;
	max-height: 100vh;
}

/* Popup */
.ol-popup {
	position: absolute;
	background: white;
	padding: 12px;
	border-radius: 8px;
	border: 1px solid #ccc;
	min-width: 180px;
	max-width: 90vw;
	font-size: 14px;
	box-shadow: 0 2px 8px rgba(0, 0, 0, 0.2);
	z-index: 9999;
	word-wrap: break-word;
}

/* popup elements */
.popup-title {
	margin: 0 0 6px 0;
	font-size: 16px;
}
.popup-details p {
	margin: 4px 0;
}
.popup-actions {
	margin-top: 10px;
	text-align: right;
}

/* focus styles for accessibility */
.btn-navigate:focus,
.popup-title:focus,
#nav-panel:focus {
	outline: 3px solid #005fcc;
	outline-offset: 2px;
}

/* Navigation Panel */
.nav-panel {
	position: fixed;
	top: 20px;
	right: 20px;
	width: 280px;
	max-height: 80vh;
	background: white;
	border-radius: 10px;
	padding: 15px;
	overflow-y: auto;
	box-shadow: 0 4px 12px rgba(0,0,0,0.2);
	z-index: 9999;
	transition: transform 0.3s ease;
}

/* Toggle button for nav panel */
.toggle-nav {
	width: 100%;
	margin-bottom: 10px;
	padding: 6px;
	background: #007bff;
	color: white;
	border: none;
	border-radius: 5px;
	cursor: pointer;
}

/* OpenLayers controls */
.ol-control button {
	width: 48px;
	height: 48px;
	font-size: 18px;
	padding: 0;
	margin: 4px;
	touch-action: manipulation;
	border-radius: 8px;
}

/* Responsive for mobile */
@media (max-width: 768px) {
	.nav-panel {
		width: 90%;
		right: 5%;
		top: auto;
		bottom: 20px;
		max-height: 40vh;
	}

	button, .toggle-nav {
		padding: 12px 16px;
		font-size: 18px;
	}
}
</style>
