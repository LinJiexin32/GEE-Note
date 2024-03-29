
#GEE笔记002


##Reducer
Reducer 的中文含义是“缩减器，整合器”。与筛选 Filter 相比，Reducer 虽然也有“减少”的意思，但其更多的含义在于“通过分析后获得统计信息”上。比如有 100 个苹果，Filter 处理后只剩下 80 个，而 Reducer 处理后得则可以到“平均重量”。

总之，Filter 着重于数量上的减少，而 Reducer 强调数学抽象的汇总。

reducer处理像元时受到掩膜的影响，掩膜决定像元的权重。mask值等于0的像元将被排除在reducer的计算之外。

###联合reducer 

对于同一个输入数据集，想要并行地使用两个reducer进行处理，可以使用combine方法。如：同时计算一个ImageCollection的均值和标准差。

只需要在combine的参数reducer2指定并行计算的另一个reducer，sharedinput参数设置为true。

**可以两个以上的reducer一起使用，即combine()后面再加一个combine()**
```js
// Load and filter the Sentinel-2 image collection.
var collection = ee.ImageCollection('COPERNICUS/S2')
    .filterDate('2016-01-01', '2016-12-31')
    .filterBounds(ee.Geometry.Point([-81.31, 29.90]));

print(collection)
// Reduce the collection.可以连着几个combine一块用
//这里mean setDev 和 minMax一块用了
var reducer = ee.Reducer.mean()
  .combine({     
    reducer2:ee.Reducer.stdDev(), 
    sharedInputs:true})
  .combine({
    reducer2:ee.Reducer.minMax(), 
    sharedInputs:true})

var cb_result = collection.reduce(reducer)

var max_result = collection.reduce(ee.Reducer.max())
//检查结果可以看到，output image的波段名是以reducer的名字命名的。
print(cb_result)
print(max_result)
//对比一下使用combine的max 和单独使用max ，结果是一致的
Map.centerObject(ee.Geometry.Point([-81.31, 29.90]))
Map.addLayer(cb_result.select('B2_max'),{},'cb_max')
Map.addLayer(max_result.select('B2_max'),{},'max_only')
```
###ImageCollection Reducer
![原理图](asset/Reduce_ImageCollection.png)
如果ImageCollection中的每个image只有一个波段，则output image 也只有一个波段，每个像素值代表了集合中所有图像在这个位置上的统计值。

如果每个image有多个波段，则output image 的每一个波段都代表了集合中所有的图像在该位置该波段上上的统计值。**即在每个波段上的统计结果都是独立的**

对于基本的统计信息，如 min、 max、 mean 等，ImageCollection 提供了 min ()、 max ()、 mean ()等快捷方法。它们的功能与调用 reduce ()完全相同，只是结果波段名不会以reducer的命名。

```js
// Load and filter the Sentinel-2 image collection.
var collection = ee.ImageCollection('COPERNICUS/S2')
    .filterDate('2016-01-01', '2016-12-31')
    .filterBounds(ee.Geometry.Point([-81.31, 29.90]));

print(collection)
// Reduce the collection.

var reducer = ee.Reducer.max()

var redu_max = collection.reduce(reducer)

var max_only = collection.max()
print(redu_max.bandNames())
print(max_only.bandNames())

```
**注意，使用collection.reduce(reducer)的output image 的波段名是以reducer的名称命名的，如：'B1_max'、'B1_mean'等，而collection.max()的输出结果的波段名是'B1'**


**特别要注意，通过 ImageCollection reduce生成的图像没有投影（默认设置为EPSG:4326投影，没有尺度信息）。这意味着在后续处理ImageCollection reducer的结果时，需要设置比例因子sacle**

###Image 空间reducer
####reduceRegion 
原理图：
![reduceReign](asset/Reduce_region.png)

计算image在region内像元的统计信息或直方图等。region通常是个geometry对象（多边形等，里边包含多个像元），如果是一个点的话，则只有该点处的一个像元参与计算。

**示例：计算各个波段在某区域内的均值**
```js
// Load input imagery: Landsat 7 5-year composite.
var image = ee.Image('LANDSAT/LE7_TOA_5YEAR/2008_2012');

// Load an input region: Sierra Nevada.
var region = ee.Feature(ee.FeatureCollection('EPA/Ecoregions/2013/L3')
  .filter(ee.Filter.eq('us_l3name', 'Sierra Nevada'))
  .first());

// Reduce the region. The region parameter is the Feature geometry.
var meanDictionary = image.reduceRegion({
  reducer: ee.Reducer.mean(),
  geometry: region.geometry(),
  scale: 30,
  maxPixels: 1e9
});

// The result is a Dictionary.  Print it.
print(meanDictionary);
```
**参数：reduceRegion(reducer, geometry, scale, crs, crsTransform, bestEffort, maxPixels, tileScale)**

通常只需要指定：
* reducer
* geometry：计算范围
* scale：尺度。（如果指定了scale，可以不用指定crs,和crsTransform。或者指定crs和crsTransform而不用指定前者。待我看一看有关projection和scale扥文档）
  **如果数据源有不同的scale，并且不指定scale,则尺度会默认设定为一度大小。**
* maxPixels:能传给reducer的最大像元数量，可以设的大一点，1e9,1e13之类的
* bestEffort：如果多边形在给定的尺度上包含太多像素，为了使计算能够成功，将使用一个更大的尺度进行计算，默认为false。

**通常，指定比例就足够了，并且代码的可读性更高。 Earth Engine 通过首先对region进行栅格化来确定将哪些像素输入到reducer。如果在没有 CRS （坐标参考系统）的情况下指定比例，则该region将在缩放到指定分辨率的图像的原始投影中栅格化。如果同时指定了 CRS 和比例，则region将基于它们进行栅格化。**

**当输入reducer的像素过多时，可以：**
* 增加 maxPixels
* 增大比例
* 将 bestEffort 设置为 true，这会自动计算（更大的）比例，以便不超过 maxPixels。

如果不指定 maxPixels，则使用默认值。

####reduceRegions
对于一个image上多个Regions的统计，可以使用image.reduceRegions方法。

**参数reduceRegions(collection, reducer, scale, crs, crsTransform, tileScale)**
参数和reduceRegion基本是类似的，就是需要输入一个FeatureCollection指定要统计的区域集合。

**注意：函数调用后输出的结果是也是一个FeatureCollection，就是在输入的FeatureCollection基础上，对每个feature增加了一些属性，这些属性存储了reducer的计算结果**

示例：分别计算多个县内的像元均值
```js
// Load input imagery: Landsat 7 5-year composite.
var image = ee.Image('LANDSAT/LE7_TOA_5YEAR/2008_2012');

// Load a FeatureCollection of counties in Maine.
var maineCounties = ee.FeatureCollection('TIGER/2016/Counties')
  .filter(ee.Filter.eq('STATEFP', '23'));
print('countries',maineCounties)

// Add reducer output to the Features in the collection.
var maineMeansFeatures = image.reduceRegions({
  collection: maineCounties,
  reducer: ee.Reducer.mean(),
  scale: 30,
});

// Print the first feature, to illustrate the result.
print('maineMeansFeatures',maineMeansFeatures)
print(ee.Feature(maineMeansFeatures.first()).select(image.bandNames()));
//可以看到，输出的FeatureCollection中的每个feature，属性里面都存储了该区域内各个波段的均值。
//本例中有七个波段，所以有七个属性，分别存储了七个波段的均值。
```
#### reduceNeighborhood
**邻域reducer，和卷积类似**

image.reduceNeighborhood()
2个重要参数：
* reducer
* 卷积核
```js
var texture = naipNDVI.reduceNeighborhood({
  reducer: ee.Reducer.stdDev(),
  kernel: ee.Kernel.circle(7),
});
```

### Featurecoll 属性reducer

对于FeatureCollection，想要计算某个字段的某些统计值，使用featureCollection.reduceColumns().

**参数：reduceColumns(reducer, selectors, weightSelectors)**
* reducer
* selectors：选择某个属性，或多个属性。列表
* weightSelectors：权重选择器。列表


**示例：计算中国各省级行政区面积总和**
```js

print(table.first())

var sum = table.reduceColumns(
                 {
                   reducer:ee.Reducer.sum(),
                   selectors:['Shape_Area']
                 })

var t1 = table.filter(ee.Filter.notNull(['Shape_Area'])).reduceColumns(
                 {
                   reducer:ee.Reducer.sum(),
                   selectors:['Shape_Area']
                 })
print(t1)
print(sum)
```
**注意：在对FeatureCollection的某个字段进行统计时，为了避免空值影响计算统计结果，可以使用ee.Filter.notNull(properties_list)先筛选掉有空值的feature。**

**示例：同时统计多个字段**
```js
var sum2 = table.reduceColumns(
                  {
                    reducer:ee.Reducer.sum().repeat(2),
                    selectors:['Shape_Area','Shape_Leng']
                  }
  )
print(sum2)
```

**reduceColums同时统计多个字段时（例子中对长度和面积都求和），需要在reducer参数后面增加repeat(num)方法，然后再selectors中增加字段名**
###矢量转栅格
使用FeatureCollection.reduceToImage(properties, reducer)方法，将矢量集转成栅格。

* properties: 属性名
*  reducer：对于那些空间上相交的feature，要怎么去处理属性？当没有空间上重叠的features时候，使用ee.Filter.first()即可。

**同样注意要ee.Filter.notNull消除空值**

**示例：将中国省级行政区域矢量图转为栅格image。image有两个波段，一个是面积，一个是长度。
```js
var area_band = table.filter(ee.Filter.notNull(['Shape_Area',]))
                  .reduceToImage(
                    {
                      properties:['Shape_Area'],
                      reducer:ee.Reducer.first()
                    }).rename('area')
var leng_band = table.filter(ee.Filter.notNull(['Shape_Leng',]))
                  .reduceToImage(
                    {
                      properties:['Shape_Leng'],
                      reducer:ee.Reducer.first()
                    }).rename('leng')

var result = area_band.addBands(leng_band)
print(result)

Map.addLayer(result,{},'image')
Map.centerObject(result)
```

**不用设置尺度比例，与 Earth Engine 中的所有图像输出reducer一样，比例是动态的。在这种情况下，比例对应于代码编辑器中的缩放级别。**


###Reduction 的结群
####FeatureCollection的结群

**示例：利用中国市县级行政区数据，分别计算出各省的面积。**
<https://code.earthengine.google.com/e46c3bdaef49fffa99d5c325359ff62c>

```js

var table2 = table2.map(function (feature)
                    {
                      var feature = ee.Feature(feature)
                      return feature.set('area_km2',feature.area().divide(1000*1000))
                    })
print(table2.limit(2))

var sum  = table2.reduceColumns(
                  {
                    reducer:ee.Reducer.sum().group(
                      {
                        groupField:1,
                        groupName:'prov_name'
                      }),     
                    selectors:['area_km2','provinces',],
                  })

print(sum)
```

结群reduction使用ee.Reducer.xx().group()函数.

参数groupField要指出结群的数据列的位置。比如本例中以省名'provinces'进行集群，所以需要给出'provinces'在selectors列表中的索引位置。**索引从0开始。**

groupName表示结果的属性名，本例中结群后每个group都有'prov_name'属性，结果存在该属性中。

**同样，如果有多个字段需要集群统计，则先repeat再group。即： reducer: ee.Reducer.sum().repeat(2).group()**

####Image 的区域统计

在image.reduceRegion()的基础上，进行结群统计，也叫区域统计。

**示例：利用土地利用分类数据，统计nightlight数据差异在各种土地利用类型上的均值。**

```js
// 美国矢量图
var region = ee.FeatureCollection('USDOS/LSIB_SIMPLE/2017')
  .filter(ee.Filter.eq('country_na', 'United States'));

// Load MODIS land cover categories in 2001.
var landcover = ee.Image('MODIS/051/MCD12Q1/2001_01_01')
  // Select the IGBP classification band.
  .select('Land_Cover_Type_1');
Map.addLayer(landcover,{},'landcover')
print('landcover',landcover)

// Load nightlights image inputs.
var nl2001 = ee.Image('NOAA/DMSP-OLS/NIGHTTIME_LIGHTS/F152001')
  .select('stable_lights');
var nl2012 = ee.Image('NOAA/DMSP-OLS/NIGHTTIME_LIGHTS/F182012')
  .select('stable_lights');

// Compute the nightlights decadal difference, add land cover codes.
var nlDiff = nl2012.subtract(nl2001).addBands(landcover);
print('nldiff',nlDiff)
// Grouped a mean reducer: change of nightlights by land cover category.
var means = nlDiff.reduceRegion({
  reducer: ee.Reducer.mean().group({
    groupField: 1,
    groupName: 'code',
  }),
  geometry: region.geometry(),
  scale: 1000,
  maxPixels: 1e8
});

// Print the resultant Dictionary.
print(means);
```
**请注意，groupField 是波段的索引。第一个波段是索引 0，第二个是索引 1，等等。在此示例中，groupField: 1,表示使用第二波段也就是土地利用分类数据进行结群**

###线性回归
Earth Engine 有几种使用 reducer 执行线性回归的方法：
* ee.Reducer.linearFit()
* ee.Reducer.linearRegression()
* ee.Reducer.robustLinearRegression()
* ee.Reducer.ridgeRegression()

其中，linearFit()是最简单的最小二乘一元线性回归方法。对于多个自变量和多个因变量的线性回归方法，使用ee.Reducer.linearRegression()
ee.Reducer.robustLinearRegression()
ee.Reducer.ridgeRegression()

linearRegression() 实现普通最小二乘回归 (OLS)。 robustLinearRegression() 使用基于回归残差的成本函数对数据中的离群值进行迭代去权重 (O’Leary, 1990)。 ridgeRegression() 使用 L2 正则化进行线性回归。（妈的没看懂，看了一晚上就知道个最小二乘线性回归）

**linearFit()和linearRegression()的区别在于前者只能进行一个自变量一个因变量的最小二乘线性拟合，即一元线性回归。后者可以进行对各自变量和多个因变量的最小二乘线性回归。**

**linearRegression()、robustLinearRegression()、ridgeRegression()的区别在于代价函数不同（没看懂）**

以上的reducer都可以作为**ee.ImageCollection、ee.Image、ee.FeatureCollection 和 ee.List对象的reduce()方法的参数**。 

请注意，linearRegression()、robustLinearRegression() 和 ridgeRegression() 都具有相同的输入和输出结构，但 linearFit() 需要双波段输入（先X后  Y），而 ridgeRegression() 有一个附加参数（lambda， 可选）和输出（pValue）。

####ImageCollection的线性回归

#####一元线性回归linearfit
collection中的每个image必须是双波段图像，**其中第一个波段是自变量，第二个波段是因变量。**

**示例：建立降雨量和时间的一元线性回归模型**
```js
// This function adds a time band to the image.
var createTimeBand = function(image) {
  // Scale milliseconds by a large constant to avoid very small slopes in the linear regression output.
  //将大的时间除以一个很大的常数，避免回归的斜率太小
  return image.addBands(image.metadata('system:time_start').divide(1e18));
};

// Load the input image collection: projected climate data.
var collection = ee.ImageCollection('NASA/NEX-DCP30_ENSEMBLE_STATS')
  .filter(ee.Filter.eq('scenario', 'rcp85'))
  .filterDate(ee.Date('2006-01-01'), ee.Date('2050-01-01'))
  // Map the time band function over the collection.
  .map(createTimeBand);

// Reduce the collection with the linear fit reducer.
//先自变量，后因变量
var linearFit = collection.select(['system:time_start', 'pr_mean'])
  .reduce(ee.Reducer.linearFit());

// Display the results.
Map.setCenter(-100.11, 40.38, 5);
Map.addLayer(linearFit,
  {min: 0, max: [-0.9, 8e-5, 1], bands: ['scale', 'offset', 'scale']}, 'fit');
```
**输出图像具有两个波段，'offset'和'scale',分别代表一元线性拟合直线的截距和斜率。**

#####多元线性回归

对多个自变量和因变量的情况，以linearRegression()为例。

**示例，假设有两个因变量：降水量和最高温度，以及两个自变量：常数和时间。 输入的前两个波段是“X”（独立）变量，接下来的两个波段是“Y”（因）变量。 在此示例中，首先获取回归系数，然后展平阵列图像以提取感兴趣的波段：**

```js
// This function adds a time band to the image.
var createTimeBand = function(image) {
  // Scale milliseconds by a large constant.
  return image.addBands(image.metadata('system:time_start').divide(1e18));
};

// This function adds a constant band to the image.
var createConstantBand = function(image) {
  return ee.Image(1).addBands(image);
};

// Load the input image collection: projected climate data.
var collection = ee.ImageCollection('NASA/NEX-DCP30_ENSEMBLE_STATS')
  .filterDate(ee.Date('2006-01-01'), ee.Date('2099-01-01'))
  .filter(ee.Filter.eq('scenario', 'rcp85'))
  // Map the functions over the collection, to get constant and time bands.
  .map(createTimeBand)
  .map(createConstantBand)
  // Select the predictors and the responses.
  .select(['constant', 'system:time_start', 'pr_mean', 'tasmax_mean']);

// Compute ordinary least squares regression coefficients.
var linearRegression = collection.reduce(
  ee.Reducer.linearRegression({
    numX: 2,
    numY: 2
}));

// Compute robust linear regression coefficients.
var robustLinearRegression = collection.reduce(
  ee.Reducer.robustLinearRegression({
    numX: 2,
    numY: 2
}));

// The results are array images that must be flattened for display.
// These lists label the information along each axis of the arrays.
var bandNames = [['constant', 'time'], // 0-axis variation.
                 ['precip', 'temp']]; // 1-axis variation.

// Flatten the array images to get multi-band images according to the labels.
var lrImage = linearRegression.select(['coefficients']).arrayFlatten(bandNames);
var rlrImage = robustLinearRegression.select(['coefficients']).arrayFlatten(bandNames);

// Display the OLS results.
Map.setCenter(-100.11, 40.38, 5);
Map.addLayer(lrImage,
  {min: 0, max: [-0.9, 8e-5, 1], bands: ['time_precip', 'constant_precip', 'time_precip']}, 'OLS');

// Compare the results at a specific point:
print('OLS estimates:', lrImage.reduceRegion({
  reducer: ee.Reducer.first(),
  geometry: ee.Geometry.Point([-96.0, 41.0]),
  scale: 1000
}));

print('Robust estimates:', rlrImage.reduceRegion({
  reducer: ee.Reducer.first(),
  geometry: ee.Geometry.Point([-96.0, 41.0]),
  scale: 1000
}));

```
**有太多没搞清楚的，暂且先跳过这部分**

####Image的线性回归

Image的线性回归，需要与 reduceRegion 或 reduceRegions 一起使用以对区域中的像素执行线性回归。

我的理解：假设在iamge内有波段b1，b2，要找出b1,b2波段的线性关系。需要在reduceRegion规定的区域内，选择区域内的像元，使用线性回归reducer，最小二乘拟合出两个波段的线性关系。


#####linearFit()
```js
// Define a rectangle geometry around San Francisco.
var sanFrancisco = ee.Geometry.Rectangle([-122.45, 37.74, -122.4, 37.8]);

// Import a Landsat 8 TOA image for this region.
var img = ee.Image('LANDSAT/LC08/C02/T1_TOA/LC08_044034_20140318');
```

**同样，自变量先，因变量后。这里B5是自变量，B6是因变量**
```js
 In the regression reducer, independent
// variables come first followed by the dependent variables. In this case,
// B5 (SWIR1) is the independent variable and B6 (SWIR2) is the dependent  variable.
var imgRegress = img.select(['B5', 'B6']);

// Calculate regression coefficients for the set of pixels intersecting the
// above defined region using reduceRegion with ee.Reducer.linearFit().
var linearFit = imgRegress.reduceRegion({
  reducer: ee.Reducer.linearFit(),
  geometry: sanFrancisco,
  scale: 30,
});

// Inspect the results.
print('OLS estimates:', linearFit);
print('y-intercept:', linearFit.get('offset'));
print('Slope:', linearFit.get('scale'));
```
##### linearRegression


**假设有一个多波段（b1,b2,b3）图像,想把b1,b2作为自变量，b3作为因变量，拟合出三个波段之间的线性关系。同样需要利用img.reduceRegion或img.reduceRegions与ee.Reducer.linearRegression的结合。原理与上面linearfit类似。**
y = a1\*x1 + a2\*x2 + b

```js
// Define a rectangle geometry around San Francisco.
var sanFrancisco = ee.Geometry.Rectangle([-122.45, 37.74, -122.4, 37.8]);

// Import a Landsat 8 TOA image for this region.
var img = ee.Image('LANDSAT/LC08/C02/T1_TOA/LC08_044034_20140318');

// Create a new image that is the concatenation of three images: a constant,
// the SWIR1 band, and the SWIR2 band.
var constant = ee.Image(1);
var xVar = img.select('B5');
var yVar = img.select('B6');
var imgRegress = ee.Image.cat(constant, xVar, yVar);

// Calculate regression coefficients for the set of pixels intersecting the
// above defined region using reduceRegion. The numX parameter is set as 2
// because the constant and the SWIR1 bands are independent variables and they
// are the first two bands in the stack; numY is set as 1 because there is only
// one dependent variable (SWIR2) and it follows as band three in the stack.
var linearRegression = imgRegress.reduceRegion({
  reducer: ee.Reducer.linearRegression({
    numX: 2,
    numY: 1
  }),
  geometry: sanFrancisco,
  scale: 30,
});

// Convert the coefficients array to a list.
var coefList = ee.Array(linearRegression.get('coefficients')).toList();

//结果是个字典，通过将结果转换为数组然后切出所需元素或将数组转换为列表并按索引位置选择系数来提取系数。
var b0 = ee.List(coefList.get(0)).get(0); // 
var b1 = ee.List(coefList.get(1)).get(0); // slope

// Extract the residuals.
var residuals = ee.Array(linearRegression.get('residuals')).toList().get(0);

// Inspect the results.
print('OLS estimates', linearRegression);
print('y-intercept:', b0);
print('Slope:', b1);
print('Residuals:', residuals);
```


*** 
## Filter时空属性过滤
数据集ImageCollection和FeatureCollection，一般使用filter方法筛选出特定的子集。

* 通用过滤方式：ImageCollection/FeatureCollection.filter(ee.Filter对象)
常用的ee.Filter对象如下：
```js
ee.Filter.eq() ee.Filter.neq() ee.Filter.ge() ee.Filter.gte() ee.Filter.le() 
ee.Filter.lte() ee.Filter.maxDifference()ee.Filter.stringContains()
ee.Filter.StarsWith() ee.Filter.EndWith() ee.Filter.Rangecontains() 
ee.Filter.listContains() ee.Filter.inList() ee.Filter.calendarRange() 
ee.FilterDateRangeContains() ee.Filter.dayOfYear()
ee.Filter.and() ee.Filter.or() ee.Filter.not()
```
* 具体过滤方式。如ImageCollection/FeatureCollection.filterBounds() filterDate()等。等价于ImageCollection/FeatureCollection.filter(ee.Filter.date())





### 空间过滤

#### filterBounds
***
## 循环遍历集合数据

###map()
**ImageCollection/FeatureCollection.map(function)**
map方法很简单。我们在遍历ImageCollection和FeatureCollection时，想要对集合中每个元素做相同的处理（如去云，计算NDVI等）。因此我们将这些处理封装成一个函数，然后xxxCollection.map(function)，即可用这个function处理集合中的每一个元素。
function的定义如下：
```js
function functionName(element){
  return xxx
}
```
element 是集合中的元素，map函数返回一个Collection，该Collection包含了每次function的处理结果xxx。
###iterate()
先给出iterate方法的格式，再详细解释参数的含义和用法。
**ImageCollection/FeatureCollection.iterate(function,fitst)**

**function:** 集合中的每一个元素都要调用的函数。

**first：** 数据类型一般和iterate()的返回值相同，可以理解为一个循环过程中“寄存器”。集合中的每个元素在执行了function后的结果可以存在这里，可以使每次迭代可以使中前面的迭代结果，不像map()方法那样每次迭代过程都是独立不相互影响的。

function的定义如下
```js
function(element,first_data) {}
```
element 是集合中的每一个元素，和map()中function的参数意义相同。first_data 是初始化iterate(function,first)中的fitst的结果，相当于把暂存区中的数据备份后拿来用。function的返回值会重新赋值给iterate(function,first)中的first，即把某次的迭代结果存在暂存区。

**例子：使用iterate方法循环计算FeatureCollection中各个feature面积的总和**
```js
print(table.first().propertyNames())
function addArea(feature,sum)
{
  var feature =  ee.Feature(feature)
  var sum = ee.Number(sum)
  
  var area = ee.Number(feature.get('Shape_Area'))
  return sum.add(area)
}

var sum_area = table.iterate(addArea,ee.Number(0));
print(sum_area)

var sum2 = table.aggregate_sum('Shape_Area');
print(sum2)

```

***
##数据集连接Join

连接是将两个数据集结合到一起的操作，这种操作可以分为两个部分，第一个是解决“用
什么字段连接”的问题，第二个是解决“连接之后怎么办”的问题。

首先定义一个Join对象，各种类型的连接对象见下文。
```js
// 定义一个反向连接对象
var invertedJoin = ee.Join.inverted();
```

接下来定义一个filter对象，filter指定了两个数据集之间匹配的条件。flter的类型有很多，如：equals, greaterThanOrEquals, lessThan等
```js
var filter = ee.Filter.equals({
  leftField: 'system:index',
  rightField: 'system:index'
});
```
最后调用join对象的apply()方法，参数为两个Collection以及上面定义的filter
```js
// Apply the join.
var invertedJoined = invertedJoin.apply(primary, secondary, filter);
```


### 简单连接Simple Join

**示例：对于4-5月和5-6月的两个数据集，使用简单连接筛选出重叠的数据**
```js
// Load a Landsat 8 image collection at a point of interest.
var collection = ee.ImageCollection('LANDSAT/LC08/C02/T1_TOA')
    .filterBounds(ee.Geometry.Point(-122.09, 37.42));

// Define start and end dates with which to filter the collections.
var april = '2014-04-01';
var may = '2014-05-01';
var june = '2014-06-01';
var july = '2014-07-01';

// The primary collection is Landsat images from April to June.
var primary = collection.filterDate(april, june);

// The secondary collection is Landsat images from May to July.
var secondary = collection.filterDate(may, july);

// 使用 equals filter定义两个数据集的匹配条件
var filter = ee.Filter.equals({
  leftField: 'system:index',
  rightField: 'system:index'
});

// Create the join.
var simpleJoin = ee.Join.simple();

// Apply the join.
var simpleJoined = simpleJoin.apply(primary, secondary, filter);

// Display the result.
print('Simple join: ', simpleJoined);
```
对于primary ImageCollection中的每一个元素，若它和secondary ImageCollection中的某个元素匹配（即在某个字段上满足filter中的条件），则该元素保留。**注意：简单连接只保留primary ImageCollection中的匹配的元素。** 

###反向连接Inverted Joins
Inverted Join 和 Simple Join相反。
Simple Join 是保留primary Collection中和secondary Collection相匹配的元素。
Inverted Join则保留了和secondary Collection不匹配的primary collection元素，把匹配的都丢了。

```js
// Load a Landsat 8 image collection at a point of interest.
var collection = ee.ImageCollection('LANDSAT/LC08/C02/T1_TOA')
    .filterBounds(ee.Geometry.Point(-122.09, 37.42));

// Define start and end dates with which to filter the collections.
var april = '2014-04-01';
var may = '2014-05-01';
var june = '2014-06-01';
var july = '2014-07-01';

// The primary collection is Landsat images from April to June.
var primary = collection.filterDate(april, june);

// The secondary collection is Landsat images from May to July.
var secondary = collection.filterDate(may, july);
print('primary',primary)
print('secondart',secondary)
// Use an equals filter to define how the collections match.
var filter = ee.Filter.equals({
  leftField: 'system:index',
  rightField: 'system:index'
});

// Define the join.
var invertedJoin = ee.Join.inverted();

// Apply the join.
var invertedJoined = invertedJoin.apply(primary, secondary, filter);

// Print the result.
print('Inverted join:', invertedJoined);
```

### 内部连接 Inner Join
ee.Join.inner()的返回值是一个FeatureCollection（即使两个输入数据集都是ImageCollection）。

返回的FeatureCollection中的每个feature都存储了一种匹配结果。每个feature都有两个属性，分别存储满足了匹配的一对元素（一个来自primary Collection，一个来自secondary collection）。

Each feature in the output represents a match, where the matching elements are stored in two properties of the feature. For example, feature.get('primary') is the element in the primary collection that matches the element from the secondary collection stored in feature.get('secondary'). (Different names for these properties can be specified as arguments to inner(), but ‘primary’ and ‘secondary’ are the defaults). 


当两个数据集中的元素存在一对多的关系时，输出的FeatureCollection中用多个feature表示这种关系。如：数据集A中的a元素对于数据集B中的1，2，,3元素，则输出的FeatureCollection中feature1:a-1、feature2:a-2、feature3:a-3。

两个集合中没有匹配的元素会被丢掉。

ImageCollection和FeatureCollection可以相互Inner Join。

```js
// Create the primary collection.
var primaryFeatures = ee.FeatureCollection([
  ee.Feature(null, {foo: 0, label: 'a'}),
  ee.Feature(null, {foo: 1, label: 'b'}),
  ee.Feature(null, {foo: 1, label: 'c'}),
  ee.Feature(null, {foo: 2, label: 'd'}),
]);

// Create the secondary collection.
var secondaryFeatures = ee.FeatureCollection([
  ee.Feature(null, {bar: 1, label: 'e'}),
  ee.Feature(null, {bar: 1, label: 'f'}),
  ee.Feature(null, {bar: 2, label: 'g'}),
  ee.Feature(null, {bar: 3, label: 'h'}),
]);

// Use an equals filter to specify how the collections match.
var toyFilter = ee.Filter.equals({
  leftField: 'foo',
  rightField: 'bar'
});
```
**ee.Join.inner('c1', 'c2'),参数c1,c2用来指定输出的FeatureCollection中feature的属性名。
对于一对匹配的元素，primary Collection中的元素放在feature的c1属性中，secondary Collection中的元素放在feature的c2属性中。**
```js
var innerJoin = ee.Join.inner('c1', 'c2');

// Apply the join.
var toyJoin = innerJoin.apply(primaryFeatures, secondaryFeatures, toyFilter);

// Print the result.
print('Inner join toy example:', toyJoin);
```
primaryFeatures

foo | label
:-----------: | :------:
0|a
1|b
1|c
2|d


secondaryFeatures
bar | label
:---: | :------:
1|e
1|f
2|g
3|h

**示例2：有时候MODIS影像的和QA波段分别放在不同的数据集中，可以使用innerJoin将二者提取出来，并使用map循环将二者合并。**

**注意：MODIS影像与其QA波段的成像时间是相同的，可以利用这个条件构建filter。**

```js
// Make a date filter to get images in this date range.
var dateFilter = ee.Filter.date('2014-01-01', '2014-02-01');

// Load a MODIS collection with EVI data.
var mcd43a4 = ee.ImageCollection('MODIS/MCD43A4_006_EVI')
    .filter(dateFilter);

// Load a MODIS collection with quality data.
var mcd43a2 = ee.ImageCollection('MODIS/006/MCD43A2')
    .filter(dateFilter);

// Define an inner join.
var innerJoin = ee.Join.inner();

// Specify an equals filter for image timestamps.
var filterTimeEq = ee.Filter.equals({
  leftField: 'system:time_start',
  rightField: 'system:time_start'
});

// Apply the join.
var innerJoinedMODIS = innerJoin.apply(mcd43a4, mcd43a2, filterTimeEq);

// Display the join result: a FeatureCollection.
print('Inner join output:', innerJoinedMODIS);
```
**使用map循环将MODIS影像和对应的QA波段合并。**
```js
// Map a function to merge the results in the output FeatureCollection.
var joinedMODIS = innerJoinedMODIS.map(function(feature) {
  return ee.Image.cat(feature.get('primary'), feature.get('secondary'));
});

// Print the result of merging.
print('Inner join, merged bands:', joinedMODIS);
```
###SaveAll Join
使用示例：获取与landsat成像时间在两天内的MODIS影像并存在对应的landsat影像的某个属性中

```js
// Load a primary collection: Landsat imagery.
var primary = ee.ImageCollection('LANDSAT/LC08/C02/T1_TOA')
    .filterDate('2014-04-01', '2014-06-01')
    .filterBounds(ee.Geometry.Point(-122.092, 37.42));

// Load a secondary collection: MODIS imagery.
var modSecondary = ee.ImageCollection('MODIS/006/MOD09GA')
    .filterDate('2014-03-01', '2014-07-01');

// Define an allowable time difference: two days in milliseconds.
var twoDaysMillis = 2 * 24 * 60 * 60 * 1000;

// Create a time filter to define a match as overlapping timestamps.
var timeFilter = ee.Filter.or(
  ee.Filter.maxDifference({
    difference: twoDaysMillis,
    leftField: 'system:time_start',
    rightField: 'system:time_end'
  }),
  ee.Filter.maxDifference({
    difference: twoDaysMillis,
    leftField: 'system:time_end',
    rightField: 'system:time_start'
  })
);
```
**Join.saveAll参数：ee.Join.saveAll(matchesKey, ordering, ascending, measureKey, outer)**

**matchesKey**:集合A中新增的属性，该属性存储集合B中所有匹配的元素。

ordering：在matchesKey属性中存储的元素以什么方式排序？

ascending：排序是否递增

measureKey：An optional property name used to save the measure of the join condition on each match.可选的属性，是否要给matchesKey中存储的每一个匹配的集合B中的元素增加一个属性，该属性和Join条件有关。在本例中是相差时间的毫秒数。

**outer：** 默认false，If true, primary rows without matches will be included in the result。正常情况下集合Ａ中未匹配的元素是被丢掉的，如果该选项是true，则即使不匹配也不会丢掉。在本例子中，如果landsat集合中某张影像未匹配到拍摄时间在两天内的MODIS影像，则该影像被丢弃。
```js
// Define the join.
var saveAllJoin = ee.Join.saveAll({
  matchesKey: 'terra',
  ordering: 'system:time_start',
  ascending: true,
  measureKey:'timeDiff'
});

// Apply the join.
var landsatModis = saveAllJoin.apply(primary, modSecondary, timeFilter);

// Display the result.
print('Join.saveAll:', landsatModis);
```
###Save-Best Joins
和SaveAll Joins类似，但Save-Best Joins存储的是匹配的最好的结果。其参数和SaveAll Joins也基本类似，具体查看文档。

**举例说明：上个例子中筛选出了与landsat成像时间在两天内的MODIS影像（每一幅Landsat可以匹配多幅MODIS影像，都在两天内），使用SaveBestJoin则可以筛选出成像时间最接近landsat成像时间的那一幅MODIS影像**

```js
// Load a primary collection: Landsat imagery.
var primary = ee.ImageCollection('LANDSAT/LC08/C02/T1_TOA')
    .filterDate('2014-04-01', '2014-06-01')
    .filterBounds(ee.Geometry.Point(-122.092, 37.42));

// Load a secondary collection: GRIDMET meteorological data
var gridmet = ee.ImageCollection('IDAHO_EPSCOR/GRIDMET');

// Define a max difference filter to compare timestamps.
var maxDiffFilter = ee.Filter.maxDifference({
  difference: 2 * 24 * 60 * 60 * 1000,
  leftField: 'system:time_start',
  rightField: 'system:time_start'
});

// Define the join.
var saveBestJoin = ee.Join.saveBest({
  matchKey: 'bestImage',
  measureKey: 'timeDiff'
});

// Apply the join.
var landsatMet = saveBestJoin.apply(primary, gridmet, maxDiffFilter);

// Print the result.
print(landsatMet);
```
**SaveAll 和 SaveBest 命令都是将符合条件的图像放到“左数据集”的属性中，而内部连接 InnerJoin 则是根据筛选条件形成一一对应的数据集和。因此，SaveAll 和 SaveBest 命令适合于分析属性，而内部连接 InnerJoin 则更适合于提取数据。**

###空间连接 SpatialJoin
空间连接主要使用两个filter
* ee.Filter.withinDistance
* ee.Filter.intersects

参数如下： 
```js
var distFilter = ee.Filter.withinDistance({
  distance: 100000,
  leftField: '.geo',
  rightField: '.geo',
  maxError: 10
});
var spatialFilter = ee.Filter.intersects({
  leftField: '.geo',
  rightField: '.geo',
  maxError: 10
});
```
maxError:最大重投影误差
**注意：空间筛选器 ee.Filter.withinDistance和intersects的参数中.geo 代表的是数据的Geometry。**

**示例：筛选出Yosemite National Park附近100km内的发电厂**
```js {.line-numbers}
// Load a primary collection: protected areas (Yosemite National Park).
var primary = ee.FeatureCollection("WCMC/WDPA/current/polygons")
  .filter(ee.Filter.eq('NAME', 'Yosemite National Park'));

// Load a secondary collection: power plants.
var powerPlants = ee.FeatureCollection('WRI/GPPD/power_plants');

// Define a spatial filter, with distance 100 km.
var distFilter = ee.Filter.withinDistance({
  distance: 100000,
  leftField: '.geo',
  rightField: '.geo',
  maxError: 10
});

// Define a saveAll join.
var distSaveAll = ee.Join.saveAll({
  matchesKey: 'points',
  measureKey: 'distance'
});

// Apply the join.
var spatialJoined = distSaveAll.apply(primary, powerPlants, distFilter);

// Print the result.
print(spatialJoined);
```

**示例：统计美国每个州的发电厂数量**
```js{.line-numbers}
// Load the primary collection: US state boundaries.
var states = ee.FeatureCollection('TIGER/2018/States');
print('states',states)
// Load the secondary collection: power plants.
var powerPlants = ee.FeatureCollection('WRI/GPPD/power_plants');

// Define a spatial filter as geometries that intersect.
var spatialFilter = ee.Filter.intersects({
  leftField: '.geo',
  rightField: '.geo',
  maxError: 10
});

// Define a save all join.
var saveAllJoin = ee.Join.saveAll({
  matchesKey: 'power_plants',
});

// Apply the join.
var intersectJoined = saveAllJoin.apply(states, powerPlants, spatialFilter);

// Add power plant count per state as a property.
intersectJoined = intersectJoined.map(function(state) {
  // Get "power_plant" intersection list, count how many intersected this state.
  var nPowerPlants = ee.List(state.get('power_plants')).size();
  // Return the state feature with a new property: power plant count.
  return state.set('n_power_plants', nPowerPlants);
});

// Make a bar chart for the number of power plants per state.
var chart = ui.Chart.feature.byFeature(intersectJoined, 'NAME', 'n_power_plants')
  .setChartType('ColumnChart')
  .setSeriesNames({n_power_plants: 'Power plants'})
  .setOptions({
    title: 'Power plants per state',
    hAxis: {title: 'State'},
    vAxis: {title: 'Frequency'}});

// Print the chart to the console.
print(chart);
```

***
## 公共库和代码调用

和python中的importt类似，提高代码复用率。实现模块化，减少代码冗余。

###公共库代码编写

```javascript{.line-numbers}
function L8_SR_NDVI(image){
  var nir = image.select('SR_B5');
  var red = image.select('SR_B4');
  return image.expression("(B5-B4)/(B5+B4)", {"B5":nir,"B4":red}).rename("NDVI");
}
exports.L8_SR_NDVI = L8_SR_NDVI;
```
**exports.导出函数名 = 库中定义的函数名;** 使用export导出，两个函数名可以不同。

###公共库代码调用

```javascript{.line-numbers}
var lib = require('users/2018302060260/TrainingforGEE:lib/L8_SR_Process');
var dataset2 = dataset.map(lib.L8_ApplyScaleFactors);
```
**调用：require('库名:文件路径')**
库名：users/2018302060260/TrainingforGEE
文件路径：lib/L8_SR_Process
![公共库路径](./asset/公共库路径.png)


###Export

####单张Image导出
```JavaScript
Export.image.toDrive({
    image: image,
    description: fileName,      //任务描述
    folder: folder,              //文件夹
    fileNamePrefix: fileName,    //文件名
    scale: 250,                   //尺寸
    region: region,               // 区域
    maxPixels: 1e13
  });
```
#### ImageCollection导出

**这里是假定每个ImageCollection中的Image都有一个属性‘year’，将year作为文件名导出**
```javascript
// 设置导出文件夹的名称
var folder = 'summerStatsNDVI_folder';
// 获取待导出的图像集合
var collection = summerStats
// 定义一个函数，将图像导出到Google Drive中
var exportImage = function(image, fileName) {
  Export.image.toDrive({
    image: image,
    description: fileName.getInfo(),
    folder: folder,
    fileNamePrefix: fileName.getInfo(),
    scale: 250,
    region: region,
    maxPixels: 1e13
  });
};

// 获取图像集合中所有的图像
var images = collection.toList(collection.size());
// 对每个图像调用导出函数
var length = images.length().getInfo()

for (var i = 0; i < length; i++) 
{
  var image = ee.Image(images.get(i));
  // 获取图像的年份属性
  var year = ee.Number(image.get('year')).toInt();
  // 定义导出文件名，格式为 "year.tif"
  var fileName = ee.String(year);
  // 导出图像
  exportImage(image, fileName);
}

```



