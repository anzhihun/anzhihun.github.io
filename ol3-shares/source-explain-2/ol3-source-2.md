title: OpenLayers 3 源码那些事（下）
speaker: 扯淡大叔
url: http://weilin.me/ol3-shares/source-explain-2/index.html
transition: slide2
theme: blue
files: /js/demo.js,/css/demo.css
date: 2016年6月1日

[slide]

# OpenLayers 3 源码那些事（下）
## Source & Layer

<br>
### 演讲者：扯淡大叔
### QQ: 11364382

<br>
#### 2016-06-01

[slide]

# Source三大类
---
* 单张图片 `ol.source.Image`
* 瓦片图片 `ol.source.Tile`
* 矢量图形 `ol.source.Vector`

[slide]

# 类图 

![三大source类结构](/images/source-types.png)

[slide]

# 单张图片 {:&.flexbox.vleft}

目前相关的类有：
* `ol.source.ImageArcGISRest `
* `ol.source.ImageCanvas` <- `ol.source.ImageVector` [Image Vector Layer](http://openlayers.org/en/latest/examples/image-vector-layer.html)
* `ol.source.ImageMapGuide`
* `ol.source.ImageStatic`
* `ol.source.ImageWMS` [Single Image WMS](http://openlayers.org/en/latest/examples/wms-image.html)
* `ol.source.Raster` [Raster Source](http://openlayers.org/en/latest/examples/raster.html)

把他们再细分一下, ArcGIS, WMS, MapGuide, Static可以分为一类。Canvas和Raster可分为另一类。

[slide]

```javascript
ol.source.Image.prototype.getImage = function(extent, resolution, pixelRatio, projection) {
  var sourceProjection = this.getProjection();
  if (!ol.ENABLE_RASTER_REPROJECTION ||
      !sourceProjection ||
      !projection ||
      ol.proj.equivalent(sourceProjection, projection)) {
    if (sourceProjection) {
      projection = sourceProjection;
    }
    return this.getImageInternal(extent, resolution, pixelRatio, projection);
  } else {

```

[slide]
```javascript
    if (this.reprojectedImage_) {
      if (this.reprojectedRevision_ == this.getRevision() &&
          ol.proj.equivalent(
              this.reprojectedImage_.getProjection(), projection) &&
          this.reprojectedImage_.getResolution() == resolution &&
          this.reprojectedImage_.getPixelRatio() == pixelRatio &&
          ol.extent.equals(this.reprojectedImage_.getExtent(), extent)) {
        return this.reprojectedImage_;
      }
      this.reprojectedImage_.dispose();
      this.reprojectedImage_ = null;
    }

    this.reprojectedImage_ = new ol.reproj.Image(
        sourceProjection, projection, extent, resolution, pixelRatio,
        function(extent, resolution, pixelRatio) {
          return this.getImageInternal(extent, resolution,
              pixelRatio, sourceProjection);
        }.bind(this));
    this.reprojectedRevision_ = this.getRevision();

    return this.reprojectedImage_;
  }
};
```


[slide]

# Image Source的核心

## `getImageInternal`

基本流程： 是否已经加载 -> 否则根据范围加载 -> 准备好各种参数 -> 构造请求用的url -> 最后load 
简单来说，就是对现有服务请求的封装。
  
# Canvas和Raster Source的不同

## 在前端绘制

`ImageCanvas`和`Raster`都是直接在前端用Canvas绘制，相对来说更加复杂，但从代码看，也是比较取巧的。

[slide]
ImageCanvas:
```javascript
extent = extent.slice();
  ol.extent.scaleFromCenter(extent, this.ratio_);
  var width = ol.extent.getWidth(extent) / resolution;
  var height = ol.extent.getHeight(extent) / resolution;
  var size = [width * pixelRatio, height * pixelRatio];

  var canvasElement = this.canvasFunction_(
      extent, resolution, pixelRatio, size, projection);
  if (canvasElement) {
    canvas = new ol.ImageCanvas(extent, resolution, pixelRatio,
        this.getAttributions(), canvasElement);
  }
  this.canvas_ = canvas;
```
[slide]
Raster:
```javascript
  var frameState = this.updateFrameState_(currentExtent, resolution, projection);

  var imageCanvas = new ol.ImageCanvas(
      currentExtent, resolution, 1, this.getAttributions(), canvas,
      this.composeFrame_.bind(this, frameState));

  this.renderedImageCanvas_ = imageCanvas;
```

[slide]
`ol.source.Raster.prototype.composeFrame_`:
```javascript
  for (var i = 0; i < len; ++i) {
    var imageData = ol.source.Raster.getImageData_(
        this.renderers_[i], frameState, frameState.layerStatesArray[i]);
    if (imageData) {
      imageDatas[i] = imageData;
    } else {
      // image not yet ready
      return;
    }
  }

  var data = {};
  this.dispatchEvent(new ol.source.RasterEvent(
      ol.source.RasterEventType.BEFOREOPERATIONS, frameState, data));

  this.worker_.process(imageDatas, data,
      this.onWorkerComplete_.bind(this, frameState, callback));
```

[slide]

# 瓦片图片 {:&.flexbox.vleft}
相对于单张图片而言，瓦片图片需要更加精细化的控制，方能把多张小图片拼成一张大的图片。从类图可以看到，用到了
`tileGrid`,`tileCache`。 由于在线网页地图图源，基本上都是瓦片，再加上各自为政，从而出现了下面相当多的瓦片图源类：

![Source Tile](/images/source-tile.png)

[slide]

# 各种`ol.source.TileImage`
---
* `ol.source.BingMaps`  [BingMap Dev](https://msdn.microsoft.com/en-us/library/ff428643.aspx) [Bing Maps](http://openlayers.org/en/latest/examples/bing-maps.html)
* `ol.source.TileArcGISRest` [Tiled ArcGIS](http://openlayers.org/en/latest/examples/arcgis-tiled.html)
* `ol.source.TileJSON` [TileJSON](http://openlayers.org/en/latest/examples/tilejson.html)
* `ol.source.TileWMS` [Tiled WMS](http://openlayers.org/en/latest/examples/wms-tiled-wrap-180.html)
* `ol.source.WMTS` [WMTS](http://openlayers.org/en/latest/examples/wmts.html)
* `ol.source.XYZ` [XYZ](http://openlayers.org/en/latest/examples/xyz.html)
* `ol.source.Zoomify` [Zoomify](http://openlayers.org/en/latest/examples/zoomify.html)

以上加载瓦片的流程都是一样的，只是针对各种瓦片的请求进行封装。 `getTile` -> `getTileInternal` -> `createTile_`

[slide]

# `ol.source.VectorTile`

`getTile` -> `new ol.VectorTile`

[slide]

# 矢量图片  `ol.source.Vector`
矢量图存在很多种类型，OpenLayers 3把他们都封装在了`format`这个概念里面了。

```javascript
  if (options.loader !== undefined) {
    this.loader_ = options.loader;
  } else if (this.url_ !== undefined) {
    goog.asserts.assert(this.format_ !== undefined,
        'format must be set when url is set');
    // create a XHR feature loader for "url" and "format"
    this.loader_ = ol.featureloader.xhr(this.url_, this.format_);
  }
```

[slide]
```javascript
ol.featureloader.xhr = function(url, format) {
  return ol.featureloader.loadFeaturesXhr(url, format,
      /**
       * @param {Array.<ol.Feature>} features The loaded features.
       * @param {ol.proj.Projection} dataProjection Data projection.
       * @this {ol.source.Vector}
       */
      function(features, dataProjection) {
        this.addFeatures(features);
      }, /* FIXME handle error */ ol.nullFunction);
};
```

[slide]
# 关键代码：
```javascript
xhr.onload = function(event) {
  if (xhr.status >= 200 && xhr.status < 300) {
    var type = format.getType();
    /** @type {Document|Node|Object|string|undefined} */
    var source;
    if (type == ol.format.FormatType.JSON ||
        type == ol.format.FormatType.TEXT) {
      source = xhr.responseText;
    } else if (type == ol.format.FormatType.XML) {
      source = xhr.responseXML;
      if (!source) {
        source = ol.xml.parse(xhr.responseText);
      }
    } else if (type == ol.format.FormatType.ARRAY_BUFFER) {
      source = /** @type {ArrayBuffer} */ (xhr.response);
    } else {
      goog.asserts.fail('unexpected format type');
    }
    if (source) {
      success.call(this, format.readFeatures(source,
          {featureProjection: projection}),
          format.readProjection(source));
    } else {
      goog.asserts.fail('undefined or null source');
    }
  } else {
    failure.call(this);
  }
}.bind(this);
```

[slide]
# 各种format
![formater](/images/ol-format.png)

`readFeatures`

[slide]
# Layer 同 Source的关系
![layer与source的关系](/images/ol_layer_base.png)

[slide]
#layer何时需要使用source？

`prepareFrame`的时候需要

`ol.renderer.canvas.ImageLayer.prototype.prepareFrame` 
```javascript 
image = imageSource.getImage(
    renderedExtent, viewResolution, pixelRatio, projection);
if (image) {
  var loaded = this.loadImage(image);
  if (loaded) {
    this.image_ = image;
  }
}
```

[slide]

`ol.renderer.canvas.TileLayer.prototype.prepareFrame`

```javascript
var tileLayer = this.getLayer();
var tileSource = tileLayer.getSource();
...
var renderables = [];
var i, ii, currentZ, tileCoordKey, tilesToDraw;
for (i = 0, ii = zs.length; i < ii; ++i) {
  currentZ = zs[i];
  tilesToDraw = tilesToDrawByZ[currentZ];
  for (tileCoordKey in tilesToDraw) {
    tile = tilesToDraw[tileCoordKey];
    if (tile.getState() == ol.TileState.LOADED) {
      renderables.push(tile);
    }
  }
}
this.renderedTiles = renderables;
```

[slide]
`ol.renderer.canvas.VectorLayer.prototype.prepareFrame`
```javascript
var vectorLayer = /** @type {ol.layer.Vector} */ (this.getLayer());
goog.asserts.assertInstanceof(vectorLayer, ol.layer.Vector,
    'layer is an instance of ol.layer.Vector');
var vectorSource = vectorLayer.getSource();
...
vectorSource.loadFeatures(extent, resolution, projection);
...
if (vectorLayerRenderOrder) {
  /** @type {Array.<ol.Feature>} */
  var features = [];
  vectorSource.forEachFeatureInExtent(extent,
      /**
       * @param {ol.Feature} feature Feature.
       */
      function(feature) {
        features.push(feature);
      }, this);
  features.sort(vectorLayerRenderOrder);
  features.forEach(renderFeature, this);
} else {
  vectorSource.forEachFeatureInExtent(extent, renderFeature, this);
}
```

[slide]
# 大体结构
<img src="/images/layer-source-struct.png"/>

[slide] 
#打赏 
打赏费用我将全部投入到webgis 3D引擎众筹项目
<br>
<img src="/images/1rmb.jpg" style="width:256px;height:256px;"/><img src="/images/5rmb.jpg" style="width:256px;height:256px;"/><img src="/images/10rmb.jpg" style="width:256px;height:256px;"/>
<img src="/images/15rmb.jpg" style="width:256px;height:256px;"/>

[slide]

# 谢谢大家！
### by 扯淡大叔