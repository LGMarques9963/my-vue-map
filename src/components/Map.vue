<template>
  <div class="map-wrap">
    <a href="https://www.maptiler.com" class="watermark"><img src="https://api.maptiler.com/resources/logo.svg"
        alt="MapTiler logo" /></a>
    <div class="map" ref="mapContainer"></div>

  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted, markRaw } from 'vue';
import { Map, NavigationControl, Marker, MapStyle, MaptilerTerrainControl } from 'maplibre-gl';

const mapContainer = ref(null);
const apiKey = ref('ysmbYpTRQA8zuUSMfopO');
const routesData = ref([]);


const initialState = {
  lng: 139.753,
  lat: 35.6844,
  zoom: 2,
};

var map = ref(null);

const loadTurfScript = async () => {
  await new Promise((resolve) => {
    const script = document.createElement('script');
    script.src = 'https://api.tiles.mapbox.com/mapbox.js/plugins/turf/v2.0.0/turf.min.js';
    script.charset = 'utf-8';
    script.onload = resolve;
    document.head.appendChild(script);
  });
};

const createMap = () => {
  map.value = new Map({
    container: mapContainer.value,
    style: `https://api.maptiler.com/maps/streets-v2/style.json?key=${apiKey.value}`,
    center: [initialState.lng, initialState.lat],
    zoom: initialState.zoom,
  });

  map.value.on('load', () => {
    map.value.addControl(new NavigationControl(), 'top-left');
  });
};

const fetchRoutes = () => {
  const fakeRespose = {
    "routes": [
      {
        "origin": [-43.1729, -22.9068],
        "destination": [-77.032, -38.913]
      },
      {
        "origin": [-122.414, 37.776],
        "destination": [-77.032, 38.913]
      },
    ]
  }
  routesData.value = fakeRespose.routes;
};

const addRoutesToMap = () => {
  routesData.value.forEach((route) => {
    const origin = route.origin;
    const destination = route.destination;
    const r = {
      type: 'FeatureCollection',
      features: [
        {
          type: 'Feature',
          geometry: {
            type: 'LineString',
            coordinates: [origin, destination],
          },
        },
      ],
    };


    const routeLayerID = 'route' + Math.random().toString(36).substr(2, 9);
    const pointLayerID = 'point' + Math.random().toString(36).substr(2, 9);
    const routeSourceID = 'routeSource' + Math.random().toString(36).substr(2, 9);
    const pointSourceID = 'pointSource' + Math.random().toString(36).substr(2, 9);
    const lineDistance = turf.lineDistance(r.features[0], 'kilometers');

    let arc = [];
    const steps = 500;

    for (let i = 0; i < lineDistance; i += lineDistance / steps) {
      const segment = turf.along(r.features[0], i, 'kilometers');
      arc.push(segment.geometry.coordinates);
    }

    r.features[0].geometry.coordinates = arc;

    const midpoint = turf.along(r.features[0], lineDistance / 2, 'kilometers').geometry.coordinates;

    const point = {
      type: 'FeatureCollection',
      features: [
        {
          type: 'Feature',
          properties: {},
          geometry: {
            type: 'Point',
            coordinates: midpoint,
          },
        }
      ],
    };

    new Marker()
      .setLngLat(origin)
      .addTo(map.value);
    
    new Marker()
      .setLngLat(destination)
      .addTo(map.value);

    map.value.addSource(routeSourceID, {
      type: 'geojson',
      data: r,
    });

    map.value.addSource(pointSourceID, {
      type: 'geojson',
      data: point,
    });

    map.value.addLayer({
      id: routeLayerID,
      source: routeSourceID,
      type: 'line',
      paint: {
        'line-color': '#007cbf',
        'line-width': 2,
      },
    });

    map.value.addLayer({
      id: pointLayerID,
      source: pointSourceID,
      type: 'symbol',
      layout: {
        'icon-image': 'airport',
        'icon-rotate': ['get', 'bearing'],
        'icon-rotation-alignment': 'auto',
        'icon-allow-overlap': true,
        'icon-ignore-placement': true,
      },
    });

  })
}
onMounted(async () => {
  await loadTurfScript();
  createMap();
  map.value.on('load', () => {
    fetchRoutes();
    setTimeout(() => {
      addRoutesToMap();
    }, 500);
    // addRoutesToMap();
  });
}),


  onUnmounted(() => {
    map.value?.remove();
  });
</script>


<style scoped>
@import '~maplibre-gl/dist/maplibre-gl.css';

.map-wrap {
  position: relative;
  width: 100%;
  height: calc(100vh - 77px);
  /* calculate height of the screen minus the heading */
}

.map {
  position: absolute;
  width: 100%;
  height: 100%;
}

.watermark {
  position: absolute;
  left: 10px;
  bottom: 10px;
  z-index: 999;
}
</style>