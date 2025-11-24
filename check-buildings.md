# Building Data Loading Check

## ✅ Verified Components

1. **GeoJSON File**: `public/tulsa-building-footprints.geojson`
   - ✅ File exists and is valid JSON
   - ✅ Contains 9,427 building features
   - ✅ Features have MultiPolygon geometries with valid coordinates
   - ✅ Coordinates are in correct format: [lng, lat]

2. **Frontend Code**:
   - ✅ `getBuildings()` function loads the file correctly
   - ✅ Building layer is configured with `type: 'fill'`
   - ✅ Paint options are set: `fill-color: #666666`, `fill-opacity: 1`
   - ✅ Layer ID: `1920-building-layer`
   - ✅ Source ID: `1920-building-source`
   - ✅ Layer is excluded from year filtering (in `YearExemptLayers`)

3. **Component Setup**:
   - ✅ `DynamicGeoJsonLayer` component handles fill layers
   - ✅ Layer is conditionally rendered based on data availability
   - ✅ Map initialization calls `getBuildings()` in `handleMapCreated`

## 🔍 How to Verify Buildings Are Loading

### Method 1: Browser Console
1. Open the app in your browser (http://localhost:5173)
2. Open Developer Tools (F12)
3. Check the Console tab for:
   - ✅ No errors about loading `tulsa-building-footprints.geojson`
   - ✅ Check Network tab - verify the file loads (status 200)
4. Check if buildings are visible on the map (gray polygons)

### Method 2: Check Map Layers
1. Open browser console
2. Run: `map.getLayer('1920-building-layer')` (should return layer object)
3. Run: `map.getSource('1920-building-source')` (should return source object)
4. Check layer visibility: `map.getLayoutProperty('1920-building-layer', 'visibility')`

### Method 3: Verify Data Loaded
In browser console:
```javascript
// Check if building data is loaded
console.log(window.buildingData); // If exposed
// Or check Vue component state in Vue DevTools
```

## 🐛 Potential Issues

1. **File Not Found**: 
   - Check if `BASE_URL` is correct (should be `/` in dev)
   - Verify file is in `public/` directory

2. **Layer Not Visible**:
   - Buildings might be outside current map viewport
   - Check map bounds/zoom level
   - Verify paint opacity isn't 0

3. **Geometry Type Mismatch**:
   - Buildings use `MultiPolygon` - ensure map supports this
   - Check if coordinates are in correct CRS (should be WGS84)

4. **Timing Issues**:
   - Buildings load after map creation
   - Check if `mbMap.value` is set before `getBuildings()` runs

## 📝 Quick Test

Add this to `getBuildings()` function to verify loading:
```javascript
async function getBuildings() {
  try {
    console.log('Loading buildings from:', `${import.meta.env.BASE_URL}tulsa-building-footprints.geojson`);
    const response = await fetch(`${import.meta.env.BASE_URL}tulsa-building-footprints.geojson`);
    const data = await response.json();
    console.log('Buildings loaded:', data.features.length, 'features');
    building1920GeoJSON.value = {
      type: 'geojson',
      data: data
    };
    building1920GeoJSON.value.data.id = '1920-building-source';
    console.log('Building GeoJSON set:', building1920GeoJSON.value);
  } catch (error) {
    console.error('Error loading buildings data:', error);
  }
}
```

