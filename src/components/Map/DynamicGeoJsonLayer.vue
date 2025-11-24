<script setup>
  import { computed, onMounted, onUnmounted, ref, inject } from 'vue';
  import { MglGeojsonLayer, MglPopup } from 'vue-mapbox3';
  import FeatureModal from '@Modals/FeatureModal.vue';
  import utils from '@utils/utils.js';
import DetailDrawer from '../Utility/DetailDrawer.vue';

  const props = defineProps({
    geojson: {
      type: Object,
      required: true,
      validator: (val) =>
        val && typeof val === 'object' &&
        val.type === 'geojson' &&
        val.data && typeof val.data === 'object' &&
        val.data.type === 'FeatureCollection' &&
        Array.isArray(val.data.features) &&
        val.data.features.length >= 0,
    },
    type: {
      type: String,
      required: true,
      validator: val => ['fill', 'line', 'circle', 'symbol'].includes(val)
    },
    paint: {
      type: Object,
      default: () => ({}),
      validator: val => typeof val === 'object' &&
        Object.keys(val).length > 0 &&
        Object.values(val).every(value => typeof value === 'string' ||
          typeof value === 'number'||
          typeof value === 'boolean')
    },
    layout: {
      type: Object,
      default: () => ({}),
      validator: val => typeof val === 'object' &&
        Object.keys(val).length > 0 &&
        Object.values(val).every(value => typeof value === 'string' ||
          typeof value === 'boolean')
    },
    layerId: {
      type: String,
      default: () => `layer-${Math.random().toString(36).slice(2)}`
    },
    filterYear: {
      type: String,
      default: ''
    },
    map: {
      type: Object,
      required: true,
      validator: val => val && typeof val === 'object' && 'on' in val && 'off' in val
    },
    featureFormatter: {
      type: Function,
      default: (feature) => feature
    },
    searchTerm: {
      type: String,
      default: ''
    }
  })

  const mapboxgl = inject('mapboxgl');

  defineExpose({
    fitMapToMarkers,
  });

  function fitMapToMarkers() {
    const bounds = new mapboxgl.LngLatBounds();

    // Get the features from the source
    const features = props.map.querySourceFeatures(props.geojson.data.id, {
      sourceLayer: props.layerId // If using vector tiles, specify the source layer
    });

    // Extend the bounds for each feature
    for (const feature of features) {
      if (feature.geometry.type === 'Point') {
        bounds.extend(feature.geometry.coordinates);
      }
    }

    // Fit the map to the calculated bounds
    props.map.fitBounds(bounds, {
      padding: 200
    });
  }

  const clickedfeature = ref({
    id: null,
    properties: {},
    geometry: { type: 'Point', coordinates: [0, 0] }
  });

  const detailRef = ref(null);
  const showDrawer = ref(false);
  const hasValidFeature = ref(false);
  let mouseMoveHandler = null;

  // Conditionally apply filter based on string year
  const layerDefinition = computed(() => {
    const includeSearch = props.layerId.includes("search");
    const YearExemptLayers = ['1920-burned-area-layer', 'poi-layer', '1920-street-layer', '1920-building-layer'];
    const hasYear = props.filterYear && utils.isYear(props.filterYear);
    const hasSearchTerm = !!props.searchTerm && includeSearch;

    const filterParts = ['all'];

    if (hasYear && !YearExemptLayers.includes(props.layerId)) {
      filterParts.push(['==', ['get', 'year'], props.filterYear === "" ? "" : Number.parseInt(props.filterYear)]);
    }

    if (hasSearchTerm) {
      filterParts.push(['in', props.searchTerm.toLowerCase(), ['get', 'searchable_text']]);
    }

    const filter = filterParts;

    return {
      id: props.layerId,
      type: props.type,
      paint: props.paint,
      layout: props.layout,
      filter: filter,
    }
  })


// Popup state
const popupCoords = ref(null);
const popupProps = ref(null);




  onMounted(() => {
    if (!props.map) return;
    
    // Wait for layer to be added, then attach click handler and cursor styling
    const attachHandler = () => {
      if (props.map.getLayer(props.layerId)) {
        props.map.on('click', props.layerId, handleClick);
        
        // Add pointer cursor on hover over this layer
        mouseMoveHandler = (e) => {
          const features = props.map.queryRenderedFeatures(e.point, {
            layers: [props.layerId]
          });
          if (features.length > 0) {
            // Set pointer cursor on both container and canvas to override grab cursor
            const container = props.map.getCanvasContainer();
            const canvas = props.map.getCanvas();
            container.style.cursor = 'pointer';
            if (canvas) {
              canvas.style.cursor = 'pointer';
            }
          }
        };
        
        // Attach mousemove handler
        props.map.on('mousemove', mouseMoveHandler);
      } else {
        setTimeout(attachHandler, 100);
      }
    };
    attachHandler();
  })

  onUnmounted(() => {
    if (props.map) {
      props.map.off('click', props.layerId, handleClick);
      if (mouseMoveHandler) {
        props.map.off('mousemove', mouseMoveHandler);
      }
    }
  })

  function validateJsonData(json) {
    if (!json || !json.data || !json.data.features || !Array.isArray(json.data.features)) {
      console.warn('Invalid GeoJSON data, expected an object with a features array.');
      return { type:'geojson', data: { id: json.data?.id || 'unknown', type: 'FeatureCollection', features: [] } };
    }
    if (!json.data.id || typeof json.data.id !== 'string') {
      console.error('GeoJSON data is missing a valid id');
      return;
    }
    return json;
  }

  function getCenterFromGeometry(geometry) {
    if (!geometry || !geometry.coordinates) return null;
    
    // Handle Point geometry
    if (geometry.type === 'Point') {
      return geometry.coordinates; // [lng, lat]
    }
    
    // Handle Polygon geometry - get first coordinate of first ring
    if (geometry.type === 'Polygon' && geometry.coordinates && geometry.coordinates[0]) {
      const firstRing = geometry.coordinates[0];
      if (firstRing && firstRing.length > 0) {
        return firstRing[0]; // First coordinate of first ring
      }
    }
    
    // Handle MultiPolygon geometry - get first coordinate of first polygon's first ring
    if (geometry.type === 'MultiPolygon' && geometry.coordinates && geometry.coordinates[0]) {
      const firstPolygon = geometry.coordinates[0];
      if (firstPolygon && firstPolygon[0] && firstPolygon[0].length > 0) {
        return firstPolygon[0][0]; // First coordinate of first ring of first polygon
      }
    }
    
    // Handle LineString geometry
    if (geometry.type === 'LineString' && geometry.coordinates && geometry.coordinates.length > 0) {
      const midIndex = Math.floor(geometry.coordinates.length / 2);
      return geometry.coordinates[midIndex];
    }
    
    // Handle MultiLineString geometry
    if (geometry.type === 'MultiLineString' && geometry.coordinates && geometry.coordinates[0]) {
      const firstLine = geometry.coordinates[0];
      if (firstLine && firstLine.length > 0) {
        const midIndex = Math.floor(firstLine.length / 2);
        return firstLine[midIndex];
      }
    }
    
    console.warn('Unable to extract center from geometry type:', geometry.type);
    return null;
  }

  async function handleClick(e) {
    if (!e?.features?.length || !e.features[0].properties) return;
    
    const formattedFeature = props.featureFormatter ? props.featureFormatter(e.features[0]) : e.features[0];
    
    // Enhance building features
    if (formattedFeature.geometry?.type === 'Polygon' || formattedFeature.geometry?.type === 'MultiPolygon') {
      formattedFeature.properties = {
        ...formattedFeature.properties,
        type: 'building',
        geometry_type: formattedFeature.geometry.type,
        location_id: formattedFeature.properties?.location_id || formattedFeature.properties?.RecNum || formattedFeature.id
      };
    }
    
    // Close drawer if open
    if (showDrawer.value) {
      showDrawer.value = false;
      await new Promise(resolve => setTimeout(resolve, 300));
    }
    
    // Update feature data
    if (!hasValidFeature.value) {
      clickedfeature.value = formattedFeature;
      hasValidFeature.value = true;
    } else {
      Object.assign(clickedfeature.value, formattedFeature);
      if (formattedFeature.properties) {
        Object.assign(clickedfeature.value.properties, formattedFeature.properties);
      }
    }
    
    // Fly to building
    const center = getCenterFromGeometry(clickedfeature.value.geometry);
    if (center) {
      props.map.flyTo({ center, zoom: 16, speed: 1.2 });
      await new Promise(resolve => setTimeout(resolve, 600));
    }
    
    showDrawer.value = true;
  }

  function openPopup()
  {
    const center = getCenterFromGeometry(clickedfeature.value.geometry);
    popupCoords.value = center || clickedfeature.value.geometry.coordinates;
  }
</script>

<template>
  <MglGeojsonLayer
    :layerId="layerId"
    :source-id="geojson.data.id"
    :source="validateJsonData(geojson)"
    :reactive="true"
    :layer="layerDefinition"
  />
   <!-- Popup for selected feature -->
  <!-- <MglPopup :coordinates="popupCoords" anchor="bottom" @close="popupCoords = null">
    <div @click="{ detailRef.value?.openDialog; }">
      <strong>{{ popupProps?.name || popupProps?.description }}</strong><br>
      <small>{{ popupProps?.type || 'Feature' }}</small>
    </div>
  </MglPopup> -->
  <!-- <slot name="modal" :feature="clickedfeature" />
  <slot /> -->
  <!-- Always render drawer with stable key to prevent DOM patching issues -->
  <DetailDrawer
    v-if="hasValidFeature"
    :item="clickedfeature"
    v-model="showDrawer"
    :key="'building-drawer-singleton'"/>
  <!-- <FeatureModal
    v-if="clickedfeature"
    :feature="clickedfeature"
    ref="detailRef"/> -->

</template>