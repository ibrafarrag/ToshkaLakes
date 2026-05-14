// ============================================================
// Toshka Lakes — Water Volume Estimator (Baseline 2018 vs 2026)
// ============================================================

var roi = ee.Geometry.Polygon([[
  [30.521120307456105, 23.809061435242764],
  [29.719118354331105, 23.10357538290122],
  [31.147341010581105, 22.231744735602557],
  [32.18554901839361,  23.25506757918088],
  [30.609010932456105, 23.89949200696],
  [30.521120307456105, 23.809061435242764]
]]);

Map.centerObject(roi, 8);

// ── 1. DEM — Copernicus GLO-30 ───────────────────────────────
var dem = ee.ImageCollection('COPERNICUS/DEM/GLO30')
  .filterBounds(roi)
  .select('DEM')
  .mosaic()
  .clip(roi);

// ── 2. Water Mask 2018 (Baseline — أدنى امتداد تاريخي) ───────
var water2018 = ee.ImageCollection('JRC/GSW1_4/MonthlyHistory')
  .filterBounds(roi)
  .filter(ee.Filter.calendarRange(2018, 2018, 'year'))
  .map(function(img){ return img.eq(2); })
  .sum().gt(0)
  .selfMask()
  .clip(roi);

// ── 3. Water Mask 2026 — Sentinel-2 MNDWI ───────────────────
var mndwi2026 = ee.ImageCollection('COPERNICUS/S2_SR_HARMONIZED')
  .filterBounds(roi)
  .filterDate('2025-10-01', '2026-05-01')
  .filter(ee.Filter.lt('CLOUDY_PIXEL_PERCENTAGE', 10))
  .select(['B3', 'B11'])
  .median()
  .normalizedDifference(['B3', 'B11'])
  .rename('MNDWI');

var water2026 = mndwi2026.gt(0.05).selfMask().clip(roi);

// ── 4. Baseline Elevation من DEM تحت mask 2018 (P10 = القاع) ─
var baseElev = dem.updateMask(water2018)
  .reduceRegion({
    reducer: ee.Reducer.percentile([10]),
    geometry: roi,
    scale: 30,
    maxPixels: 1e10,
    bestEffort: true
  }).getNumber('DEM');

// ── 5. Current Water Surface Elevation 2026 (P90) ────────────
var currentElev = dem.updateMask(water2026)
  .reduceRegion({
    reducer: ee.Reducer.percentile([90]),
    geometry: roi,
    scale: 30,
    maxPixels: 1e10,
    bestEffort: true
  }).getNumber('DEM');

print('Baseline Elev — 2018 P10 (m):', baseElev);
print('Surface Elev  — 2026 P90 (m):', currentElev);

// ── 6. Depth Map ─────────────────────────────────────────────
var depth = ee.Image.constant(currentElev)
  .subtract(dem)
  .updateMask(water2026)
  .max(ee.Image(0))
  .rename('depth_m');

// ── 7. Volume & Area ─────────────────────────────────────────
var pixelArea = ee.Image.pixelArea();

var stats = depth.addBands(pixelArea.rename('area'))
  .reduceRegion({
    reducer: ee.Reducer.sum(),
    geometry: roi,
    scale: 30,
    maxPixels: 1e10,
    bestEffort: true
  });

var volumeM3  = depth.multiply(pixelArea)
  .reduceRegion({
    reducer: ee.Reducer.sum(),
    geometry: roi,
    scale: 30,
    maxPixels: 1e10,
    bestEffort: true
  }).getNumber('depth_m');

var areaKM2   = stats.getNumber('area').divide(1e6);
var volumeKM3 = volumeM3.divide(1e9);

print('━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━');
print('Water Area   2026 (km²):', areaKM2);
print('Water Volume 2026 (km³):', volumeKM3);
print('Water Volume 2026  (m³):', volumeM3);
print('━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━');

// ── 8. Visualization ─────────────────────────────────────────
Map.addLayer(dem,
  {min:155, max:220, palette:['#313695','#74add1','#ffffbf','#f46d43','#a50026']},
  'DEM — Copernicus GLO-30');

Map.addLayer(water2018.visualize({palette:['#ff8800']}), {},
  'Water Extent 2018 (Baseline)', false);

Map.addLayer(water2026.visualize({palette:['#0055ff']}), {},
  'Water Mask 2026');

Map.addLayer(depth,
  {min:0, max:15, palette:['#ffffcc','#41b6c4','#225ea8','#0c2c84']},
  'Water Depth 2026 (m)');

// ── 9. Export ────────────────────────────────────────────────
Export.image.toDrive({
  image: depth.toFloat(),
  description: 'Toshka_WaterDepth_2026',
  folder: 'GEE_Exports',
  region: roi,
  scale: 30,
  crs: 'EPSG:32636',
  maxPixels: 1e10
}); 
