<template>
  <div class="map-wrap">
    <a href="https://www.maptiler.com" class="watermark"><img src="https://api.maptiler.com/resources/logo.svg"
        alt="MapTiler logo" /></a>
    <div class="map" ref="mapContainer"></div>

  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted, markRaw } from 'vue';
import { Map, NavigationControl, Marker, MapStyle } from 'maplibre-gl';

const mapContainer = ref(null);
const apiKey = ref('ysmbYpTRQA8zuUSMfopO');
const origin = ref([-122.414, 37.776]);
const destination = ref([-77.032, 38.913]);

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

    

onMounted(async () => {
  await loadTurfScript();

  map.value = new Map({
    container: mapContainer.value,
    style: `https://api.maptiler.com/maps/streets-v2/style.json?key=${apiKey.value}`,
    center: origin.value,
    zoom: initialState.zoom
  });

  const route = {
    type: 'FeatureCollection',
    features: [
      {
        type: 'Feature',
        geometry: {
          type: 'LineString',
          coordinates: [origin.value, destination.value],
        },
      },
    ],
  };

  
  const lineDistance = turf.lineDistance(route.features[0], 'kilometers');
  
  let arc = [];
  const steps = 500;
  
  for (let i = 0; i < lineDistance; i += lineDistance / steps) {
    const segment = turf.along(route.features[0], i, 'kilometers');
    arc.push(segment.geometry.coordinates);
  }
  
  route.features[0].geometry.coordinates = arc;
  
  const midpoint = turf.along(route.features[0], lineDistance / 2, 'kilometers').geometry.coordinates;
  
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


  map.value.on('load', function () {
    map.value.addControl(new NavigationControl(), 'top-left');
    
    new Marker()
      .setLngLat(origin.value)
      .addTo(map.value);

    new Marker()
      .setLngLat(destination.value)
      .addTo(map.value);

    map.value.addSource('route', {
      type: 'geojson',
      data: route,
    });

    map.value.addSource('point', {
      type: 'geojson',
      data: point,
    });

    map.value.addLayer({
      id: 'route',
      source: 'route',
      type: 'line',
      paint: {
        'line-color': '#007cbf',
        'line-width': 2,
      },
    });

    map.value.addLayer({
      id: 'point',
      source: 'point',
      type: 'symbol',
      layout: {
        'icon-image': 'airport',
        'icon-rotate': ['get', 'bearing'],
        'icon-rotation-alignment': 'auto',
        'icon-allow-overlap': true,
        'icon-ignore-placement': true,
      },
    });
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