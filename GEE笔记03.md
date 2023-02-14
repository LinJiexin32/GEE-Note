#GEE基础03
##Applies scaling factors.
###代码示例
```javascript{.line-numbers}
function applyScaleFactors(image) {
  var opticalBands = image.select('SR_B.').multiply(0.0000275).add(-0.2);
  var thermalBands = image.select('ST_B.*').multiply(0.00341802).add(149.0);
  return image.addBands(opticalBands, null, true)
              .addBands(thermalBands, null, true);
}

dataset = dataset.map(applyScaleFactors);
```

###image.addBands()详解

####addBands(srcImg, names, overwrite)
>Returns an image containing all bands copied from the first input and selected bands from the second input,optionally overwriting bands in the first image with the same name. 
The new image has the metadata and footprint from the first input image.
**从srcImage中选择波段（波段名由names指定）加入到image中，可以选择覆盖或者不覆盖**

####Arguments:
>this:dstImg (Image):
An image into which to copy bands.

>srcImg (Image):
An image containing bands to copy.

>names (List, default: null):
Optional list of band names to copy. If names is omitted, all bands from srcImg will be copied over.


>overwrite (Boolean, default: false):
If true, bands from srcImg will override bands with the same names in dstImg. Otherwise the new band will be renamed with a numerical suffix (foo to foo_1 unless foo_1 exists, then foo_2 unless it exists, etc).
**如果为true，则srcImg中的波段将覆盖dstImg中同名的波段。
为false，新波段将使用数字后缀重命名(同名波段后面加_1,如SR_B2如果不覆盖,则新波段为SR_B2_1)**。
####Returns: Image

###注意事项
**对需要进行辐射定标的图像图像，直接复制以上代码使用，修改波段名称和定标参数即可。
代码第4、5行对目标波段进行处理后，再覆盖原图像中对应的波段。这样就不会影响其他波段，如果直接return处理后的波段，那就丢掉了其他不处理的波段（如：QA波段等）。**

***
##植被指数计算（以NDVI为例）
**NDVI主要用来检测植被生长状态、植被覆盖度和消除部分辐射误差等，其取值范围-1<=NDVI<=1，负值表示地面覆盖为云、水、雪等，对可见光高反射；0表示有岩石或裸土等，NIR和R近似相等；正值，表示有植被覆盖，且随覆盖度增大而增大；**
###常见传感器的NDVI计算波段
>NDVI = (近红外波段 - 红波段) / (近红外波段 + 红波段)
Landsat8: NDVI = (band5 - band4) / (band5 + band4)
Sentinel2: NDVI = (band8 - band4) / (band8 + band4)
Modis: NDVI = (band2 - band1) / (band2 + band1)
ETM/TM: NDVI = (band4 - band3) / (band4 + band3)
>AVHRR: NDVI = (CH2 - CH1) / (CH2 + CH1)

###方法一：通过将数学公式翻译为代码直接计算
```javascript{.line-numbers}
function NDVI_V1(img) {
 var nir = img.select("SR_B5");
 var red = img.select("SR_B4");
 var ndvi = nir.subtract(red).divide(nir.add(red)).rename('ndvi');
 return ndvi;
}
```
###方法二：通过Image.expression()方法解析字符串计算。
这种方式更加灵活，在某些特殊情况下非常好用，而且非常直观。
```javascript{.line-numbers}
function NDVI_V2(img) {
 var nir = img.select("SR_B5");
 var red = img.select("SR_B4");
 var ndvi = img.expression(
   "(B5 - B4)/(B5 + B4)",
   {
     "B5": nir,
     "B4": red
   }
 ).rename('ndvi');
 return ndvi;
}
```
###方法三：直接调用Image.normalizedDifference()方法
```javascript{.line-numbers}
function NDVI_V3(img) {
 var ndvi = img.normalizedDifference(["SR_B5","SR_B4"]).rename('ndvi');
 return ndvi;
}
```
###三种计算方法区别
前两种计算方法基本没有区别。对于Image.normalizedDifference()方法，任何一个输入波段中的负像素值都会导致输出像素masked。若要避免masked，请使用 Image.expression ()计算NDVI。
###常用的NDVI可视化参数
```javascript{.line-numbers}
var visualization = {
    palette:['FFFFFF', 'CE7E45', 'DF923D', 'F1B555', 'FCD163', '99B718', '74A901',
    '66A000', '529400', '3E8601', '207401', '056201', '004C00', '023B01',
    '012E01', '011D01', '011301'],
    min: 0.0,
    max: 0.7
};
```
***

##批量去云
```js
function maskL8sr(image) {
  // Bits 3 and 5 are cloud shadow and cloud, respectively.
  var cloudShadowBitMask = 1 << 3;
  var cloudsBitMask = 1 << 5;

  // Get the pixel QA band.
  var qa = image.select('pixel_qa');

  // Both flags should be set to zero, indicating clear conditions.
  var mask = qa.bitwiseAnd(cloudShadowBitMask).eq(0)
      .and(qa.bitwiseAnd(cloudsBitMask).eq(0));

  // Return the masked image.
  return image.updateMask(mask);
}
```


###按位左移/右移

1 << 3 
// 先将1以二进制形式表示，再左移三个比特位，即000001 → 001000

1 << 5
// 将1以二进制形式表示，再左移五个比特位，即000001 → 100000

(16 >> 3);
//先将16表示为二进制形式，再右移三个比特位，即10000 → 00010

###bitwiseAnd按位与运算

情况1：
000000000000 // Image QA   
000000001000 // cloudShadowBitMask
000000000000 // 按位与运算的结果为0，所以无云阴影

情况2：
000000001000 // Image QA 
000000001000 // cloudShadowBitMask
000000001000 // 按位与运算的结果，转为十进制等于8，存在云阴影


因此
```js
var mask = qa.bitwiseAnd(cloudShadowBitMask).eq(0)
             .and(qa.bitwiseAnd(cloudsBitMask).eq(0));
```
表示**QA 中，代表云的比特位和代表云阴影的比特位都为0，** 是“干净”的条件。


**示例：选择一张云覆盖量很高的L8影像，对比除云前后的像元值**
```js
var dataset = ee.ImageCollection('LANDSAT/LC08/C02/T1_L2')
                .filterDate('2021-05-01', '2021-06-01');
var visualization = {
  bands: ['SR_B4', 'SR_B3', 'SR_B2'],
  min: 0.0,
  max: 0.3,
};

var img =  dataset.filter(ee.Filter.gt("CLOUD_COVER",40))
                  .sort("CLOUD_COVER",true)
                  .first()
                  
print(img)
Map.centerObject(img,10)
Map.addLayer(img, visualization, 'True Color (432) cloudy');

var cloudBitmask = 1 << 3
var cloudShadow = 1 << 4

var qa = img.select('QA_PIXEL')
var mask = qa.bitwiseAnd(cloudBitmask).eq(0)
             .and(qa.select('QA_PIXEL').bitwiseAnd(cloudShadow).eq(0))

var img_nocloud = img.updateMask(mask);
Map.addLayer(mask,{min:0,max:1,palette:['FFFFFF', 'CE7E45']},'mask')
Map.addLayer(img_nocloud, visualization, 'True Color (432)');

```

或者直接
```js
  // Bit 0 - Fill
  // Bit 1 - Dilated Cloud
  // Bit 2 - Cirrus
  // Bit 3 - Cloud
  // Bit 4 - Cloud Shadow
    var mask = image.select('QA_PIXEL').bitwiseAnd(parseInt('11111', 2)).eq(0);
```
**parseInt(string，radix) 是个JavaScript函数，可解析一个字符串，并返回一个整数。**
string	必需。要被解析的字符串。
radix	  可选。表示要解析的数字的基数。该值介于2 ~ 36 之间。

parseInt('11111', 2)将‘11111’解析为二进制的11111，即15

**这种写法直接就对0-4五个比特位置做了QA筛选。**
