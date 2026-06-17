// 1. Load and preprocess Landsat 8 data
function prepLandsat(image) {
  var qaMask = image.select('QA_PIXEL').bitwiseAnd(parseInt('11111', 2)).eq(0);
  var saturationMask = image.select('QA_RADSAT').eq(0);
  
  var scaleFactor = ee.Image.constant([0.0000275]); // Reflectance scale factor
  var addFactor = ee.Image.constant([-0.2]); // Reflectance offset factor
  
  var scaled = image.select('SR_B.*').multiply(scaleFactor).add(addFactor);
  
  return image.addBands(scaled, null, true)
    .updateMask(qaMask)
    .updateMask(saturationMask);
}

// 2. Load Landsat 8 Collection 2 Level 2
var landsat = ee.ImageCollection('LANDSAT/LC08/C02/T1_L2')
  .filterBounds(aoi)
  .filterDate('2023-07-01', '2023-07-31')
  .map(prepLandsat)
  .median()
  .clip(aoi);

// 3. Compute NDVI
var nir = landsat.select('SR_B5');
var red = landsat.select('SR_B4');
var ndvi = nir.subtract(red).divide(nir.add(red)).rename('NDVI');

// 4. Print NDVI Min/Max
print('NDVI Min-Max:', ndvi.reduceRegion({
  reducer: ee.Reducer.minMax(),
  geometry: aoi,
  scale: 30,
  maxPixels: 1e13
}));

// 5. Visualization Parameters
var trueColorParams = { min: 0, max: 0.3, bands: ['SR_B4', 'SR_B3', 'SR_B2'] };
var ndviParams = { min: -0.2, max: 1, palette: ['blue', 'white', 'green'] };

print('NDVI Symbology:', ndviParams);

// 6. Add Layers to Map
Map.centerObject(aoi, 12);
Map.addLayer(landsat, trueColorParams, 'True Color (RGB)');
Map.addLayer(ndvi, ndviParams, 'NDVI', false);  // Hidden by default

// 7. NDVI Legend
var legend = ui.Panel({ style: { position: 'bottom-right', padding: '8px 15px' } });

var makeLegendRow = function(color, label) {
  var colorBox = ui.Label('', { backgroundColor: color, padding: '8px', margin: '0px 8px 0px 0px' });
  var description = ui.Label(label, { margin: '0px' });
  return ui.Panel({ widgets: [colorBox, description], layout: ui.Panel.Layout.Flow('horizontal') });
};

legend.add(ui.Label('Legend', { fontWeight: 'bold', fontSize: '14px', margin: '0px 0px 4px 0px' }));
legend.add(ui.Label('NDVI', { fontWeight: 'bold' }));
legend.add(makeLegendRow('blue', 'Water (-0.2)'));
legend.add(makeLegendRow('white', 'Bare Soil (0)'));
legend.add(makeLegendRow('green', 'Vegetation (1)'));

Map.add(legend);

// 8. Export NDVI + Original Landsat Bands to Google Drive
Export.image.toDrive({
  image: ndvi,
  description: 'NDVI_Landsat8_July2023',
  scale: 30,
  region: aoi,
  maxPixels: 1e13
});

Export.image.toDrive({
  image: landsat,
  description: 'Landsat8_AllBands_July2023',
  scale: 30,
  region: aoi,
  maxPixels: 1e13
});
