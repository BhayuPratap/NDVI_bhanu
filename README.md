# NDVI_bhanu
# for different indicis analyzes. 

// Define the region of interest: Rajasthan
var rajasthan = ee.FeatureCollection('FAO/GAUL/2015/level1')
                .filter(ee.Filter.eq('ADM1_NAME', 'Rajasthan'));

// Define the time period and the monsoon months
var startDate = '2000-07-01';
var endDate = '2023-09-30';

// Load MODIS NDVI data
var modisNDVI = ee.ImageCollection('MODIS/006/MOD13Q1')
                .filterDate(startDate, endDate)
                .filterBounds(rajasthan)
                .select('NDVI');

// Function to convert 16-bit integer NDVI to 32-bit float and scale it to get original values
function scaleNDVI(image) {
  return image.toFloat().multiply(0.0001).copyProperties(image, ['system:time_start']);
}

// Apply the scaling function to the MODIS NDVI collection
var scaledNDVI = modisNDVI.map(scaleNDVI);

// Mask out non-positive NDVI values (we assume these are invalid data)
var maskedNDVI = scaledNDVI.map(function(image) {
  return image.updateMask(image.select('NDVI').gt(0));
});

// Function to calculate the average NDVI for the monsoon season
function calculateMonsoonNDVI(year) {
  var start = ee.Date.fromYMD(year, 7, 1);
  var end = ee.Date.fromYMD(year, 9, 30);
  var monsoonNDVI = maskedNDVI.filterDate(start, end).mean().set('year', year);
  return monsoonNDVI;
}

// Generate a list of years from 2000 to 2023
var years = ee.List.sequence(2000, 2023);

// Apply the monsoon NDVI calculation for each year
var yearlyNDVI = ee.ImageCollection.fromImages(years.map(function(year) {
  return calculateMonsoonNDVI(year);
}));

// Print and visualize the yearly average NDVI
print(yearlyNDVI);

var ndviVis = {min: 0, max: 0.85, palette: ['blue', 'white', 'green']};
Map.addLayer(yearlyNDVI.mean().clip(rajasthan), ndviVis, 'Mean NDVI 2000-2023');

// Calculate yearly average NDVI values
var ndviChart = ui.Chart.image.seriesByRegion({
  imageCollection: yearlyNDVI,
  regions: rajasthan,
  reducer: ee.Reducer.mean(),
  scale: 500,
  xProperty: 'year',
  seriesProperty: 'ADM1_NAME'
}).setOptions({
  title: 'Yearly Average NDVI for Monsoon (2000-2023)',
  hAxis: {title: 'Year'},
  vAxis: {title: 'Average NDVI'},
  lineWidth: 1,
  pointSize: 3
});
print(ndviChart);

// Function to calculate maximum NDVI value in Rajasthan
var maxNDVI = maskedNDVI.max();
var maxNDVIValue = maxNDVI.reduceRegion({
  reducer: ee.Reducer.max(),
  geometry: rajasthan,
  scale: 500,
});

print('Maximum NDVI in Rajasthan', maxNDVIValue);

// Create a chart to show NDVI values in descending order by year
var ndviDescendingChart = ui.Chart.image.series({
  imageCollection: yearlyNDVI,
  region: rajasthan,
  reducer: ee.Reducer.mean(),
  scale: 500,
  xProperty: 'year'
}).setOptions({
  title: 'NDVI Descending Order by Year (2000-2023)',
  hAxis: {title: 'Year'},
  vAxis: {title: 'Average NDVI'},
});
print(ndviDescendingChart);
