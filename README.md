// ---------------------------------------------
// NDVI, LST, and UHI ANALYSIS – SOUTHWEST NIGERIA
// By: Olaleye Daniel
// ---------------------------------------------

// 1. Define utility to get region from LGA names
function getRegion(lgaNames) {
  return ee.FeatureCollection("FAO/GAUL/2015/level2")
    .filter(ee.Filter.eq('ADM0_NAME', 'Nigeria'))
    .filter(ee.Filter.inList('ADM2_NAME', lgaNames))
    .union()
    .geometry();
}

// 2. Define ROIs
var akure = getRegion(['Akure South', 'Akure North']);
var ibadan = getRegion([
  'Ibadan North', 'Ibadan North-East', 'Ibadan North-West',
  'Ibadan South-East', 'Ibadan South-West'
]);
var osogbo = getRegion(['Osogbo', 'Olorunda']);
var ondo = getRegion(['Ondo East', 'Ondo West']);

var regions = [
  { name: 'Akure', geom: akure },
  { name: 'Ibadan', geom: ibadan },
  { name: 'Osogbo', geom: osogbo },
  { name: 'Ondo', geom: ondo }
];

var allGeoms = akure.union(ibadan).union(osogbo).union(ondo);
Map.centerObject(allGeoms, 8);

// 3. Display ROIs
Map.addLayer(akure, { color: 'red' }, 'Akure ROI');
Map.addLayer(ibadan, { color: 'blue' }, 'Ibadan ROI');
Map.addLayer(osogbo, { color: 'green' }, 'Osogbo ROI');
Map.addLayer(ondo, { color: 'orange' }, 'Ondo ROI');

// 4. Preprocessing
function maskLandsat(img) {
  var qa = img.select('QA_PIXEL');
  var cloud = 1 << 5;
  var shadow = 1 << 3;
  var mask = qa.bitwiseAnd(cloud).eq(0).and(qa.bitwiseAnd(shadow).eq(0));
  return img.updateMask(mask);
}

// 5. Function to compute NDVI and LST
function getDrySeasonImage(year, geom) {
  var start = ee.Date.fromYMD(year, 11, 1);
  var end = ee.Date.fromYMD(year + 1, 2, 28);
  var isL7 = year < 2014;

  var collection = isL7
    ? ee.ImageCollection("LANDSAT/LE07/C02/T1_L2")
    : ee.ImageCollection("LANDSAT/LC08/C02/T1_L2");

  var filtered = collection
    .filterBounds(geom)
    .filterDate(start, end)
    .map(maskLandsat);

  var median = filtered.median();

  return ee.Algorithms.If(
    filtered.size().eq(0),
    null,
    (function () {
      var image = ee.Image(median).clip(geom);

      var thermal = isL7 ? image.select('ST_B6') : image.select('ST_B10');
      var nir = isL7 ? image.select('SR_B4') : image.select('SR_B5');
      var red = isL7 ? image.select('SR_B3') : image.select('SR_B4');

      var ndvi = nir.subtract(red).divide(nir.add(red)).rename('NDVI');
      var lst = thermal.multiply(0.00341802).add(149).rename('LST');

      return image
        .addBands(ndvi)
        .addBands(lst)
        .set({
          'year': year,
          'system:time_start': ee.Date.fromYMD(year, 1, 1).millis()
        });
    })()
  );
}

// 6. Analysis years
var years = [2013, 2016, 2020];

// 7. Visual Layers: NDVI, LST, UHI
years.forEach(function(year) {
  regions.forEach(function(region) {
    var img = getDrySeasonImage(year, region.geom);
    if (img !== null) {
      img = ee.Image(img);
      var ndvi = img.select('NDVI');
      var lst = img.select('LST');

      var meanLst = lst.reduceRegion({
        reducer: ee.Reducer.mean(),
        geometry: region.geom,
        scale: 30,
        maxPixels: 1e13
      }).get('LST');

      var uhi = lst.subtract(ee.Number(meanLst)).rename('UHI');

      Map.addLayer(ndvi, {
        min: 0, max: 1,
        palette: ['brown', 'yellow', 'green']
      }, region.name + ' NDVI ' + year);

      Map.addLayer(lst, {
        min: 290, max: 315,
        palette: ['blue', 'yellow', 'red']
      }, region.name + ' LST ' + year);

      Map.addLayer(uhi, {
        min: 0, max: 10,
        palette: ['white', 'red']
      }, region.name + ' UHI ' + year);
    }
  });
});

// 8. NDVI & LST Time Series Charts
regions.forEach(function(region) {
  var imageList = years.map(function(year) {
    var img = getDrySeasonImage(year, region.geom);
    return ee.Algorithms.If(
      img === null,
      null,
      ee.Image(img).set({
        'system:time_start': ee.Date.fromYMD(year, 1, 1).millis(),
        'year': year
      })
    );
  });

  var filteredList = ee.List(imageList).removeAll([null]);
  var col = ee.ImageCollection.fromImages(filteredList);

  print(ui.Chart.image.series({
    imageCollection: col.select('NDVI'),
    region: region.geom,
    reducer: ee.Reducer.mean(),
    scale: 30
  }).setOptions({
    title: region.name + ' - NDVI Time Series',
    hAxis: { title: 'Year' },
    vAxis: { title: 'NDVI' },
    lineWidth: 2,
    pointSize: 4
  }));

  print(ui.Chart.image.series({
    imageCollection: col.select('LST'),
    region: region.geom,
    reducer: ee.Reducer.mean(),
    scale: 30
  }).setOptions({
    title: region.name + ' - LST Time Series',
    hAxis: { title: 'Year' },
    vAxis: { title: 'LST (Kelvin)' },
    lineWidth: 2,
    pointSize: 4
  }));
});

// 9. UHI Intensity Chart
var uhiFeatures = [];

years.forEach(function(year) {
  regions.forEach(function(region) {
    var img = getDrySeasonImage(year, region.geom);
    if (img !== null) {
      img = ee.Image(img);
      var lst = img.select('LST');

      var mean = lst.reduceRegion({
        reducer: ee.Reducer.mean(),
        geometry: region.geom,
        scale: 30,
        maxPixels: 1e13
      }).get('LST');

      var max = lst.reduceRegion({
        reducer: ee.Reducer.max(),
        geometry: region.geom,
        scale: 30,
        maxPixels: 1e13
      }).get('LST');

      var uhiVal = ee.Number(max).subtract(ee.Number(mean));

      var feature = ee.Feature(null, {
        'Region': region.name,
        'Year': year,
        'UHI': uhiVal
      });

      uhiFeatures.push(feature);
    }
  });
});

var uhiFC = ee.FeatureCollection(uhiFeatures);

print(ui.Chart.feature.groups({
  features: uhiFC,
  xProperty: 'Year',
  yProperty: 'UHI',
  seriesProperty: 'Region'
}).setChartType('LineChart').setOptions({
  title: 'Urban Heat Island Intensity (Max - Mean LST)',
  hAxis: { title: 'Year' },
  vAxis: { title: 'UHI (Kelvin)' },
  lineWidth: 2,
  pointSize: 5
}));
