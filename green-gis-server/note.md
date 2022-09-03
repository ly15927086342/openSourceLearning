# green-gis-server

[github](https://github.com/shengzheng1981/green-gis-server)

## 选择原因

如果有关注知乎 webgis 相关栏目的伙伴应该看过 shengzheng 的视频，就是介绍他的 green-gis 系列的课程，我看了他的代码仓库，也都是围绕 green-gis 拆分出来的几个部分。这个仓库是其中之一，看仓库名就了解，是一个后台服务系统。因为我还没毕业的时候就关注过这位大佬，但一直没有时间去看他的源码，现在工作了抽一些周末时间来补课。如果你对 gis 的原理感兴趣，也想自己搭建一个 gis 系统，不妨去知乎、b 站来学习下这个系列的教程。

## 整体浏览

首先我看了一下仓库的目录，非常简单，public 和 views 可以忽略，核心的文件夹就 4 个，bin、core、models、routes。

我又看了下 README.md，features 介绍了四个特性，

1. 发布 shapefile
2. 返回 feature collection 的 geojson
3. 如果是后端渲染，利用 node-canvas 返回动态的图片瓦片
4. 如果是前端渲染，根据瓦片的 x/y/z 返回 feature colleciton

核心依赖：

1. mongodb + mongoose
2. node-gdal
3. node-canvas

再看下 package.json，也非常简单，只有一个 start 命令和部分依赖，整个项目非常精简。

## bin/www

这个文件是启动文件，从 package.json 中可以看出，npm run start 执行的是 node ./bin/www

这个文件做了几件事：

1. 连接数据库 mongodb
2. 创建 http 服务，监听端口
3. 创建了 websocket，监听 stream 事件，并回传数据

亮点在于使用了 mongoose，我查了一下，mongoose 是 mongodb 的一个对象模型工具，封装了对 Document 的操作，更加便于使用。它有几个概念：

1. schema 是对表结构的定义
2. model 是由 schema 生成的模型，具有数据库操作的行为
3. entity 是由 model 创建的实体，通过 save 方法保存数据

可以看到，在 models 文件夹下，其实就是定义了不同的 schema 和对应的 model，这样很方便就能了解不同表的数据结构。

此外，在数据传输方面采用的 websocket 也是一个亮点。其中有一些小的技巧，例如，在 features 达到指定的 buffer 后才进行服务端->客户端的数据传输。这样的好处是减少了一个包的数据量，防止数据过大导致的传输失败。

## models

1. feature-class 要素类
2. label 标注
3. layer 图层
4. map 地图
5. symbol 符号

结构为： map > layer > feature-class,label,symbol

## core

### canvas

只提供了一个服务端绘制瓦片的方法 draw，这个方法有很多亮点

首先，symbol 可以从 layer 中获取，而 symbol 提供了三种渲染形式，默认是全部渲染，一种是分类渲染（category），一种是分级渲染（breaks），而用来渲染的字段放在 layer.renderer.class.field 中，美中不足的是，似乎只能用固定的字段来渲染。

其次，根据传入的 x,y,z 从数据库中，查询对应的 features

```javascript
//query features
const query = {};
query["zooms." + z + ".tileMin.tileX"] = { $lte: x };
query["zooms." + z + ".tileMin.tileY"] = { $lte: y };
query["zooms." + z + ".tileMax.tileX"] = { $gte: x };
query["zooms." + z + ".tileMax.tileY"] = { $gte: y };
const model = schema.model(layer.class.name);
const features = await model.find(query).lean();
```

这里有个点值得注意，tileMin 用的是 $lte，而 tileMax 用的是 $gte。这是因为，每个要素的瓦片索引，用的是要素空间范围的最小瓦片 tileMin 和最大瓦片 tileMax，而传入的 x,y,z 必须和这个空间范围有交集才行。因此，tileMin 和 tileMax 形成了一个矩形框，只要矩形框包含了 x,y,z，就会被查询出来。

最后，查询得到的要素集 features 需要绘制在 256\*256 的后端 canvas 画布上，不同 type 的绘制方法大同小异，都是拿到要素的坐标串，以及样式，然后借助 canvas API 进行绘。这边需要用到 convert.js 中的 lngLat2Pixel，因为每个坐标点都要转换成当前瓦片的像素坐标，才可以正确绘制。

需要注意的是，一个较大范围的要素，不太可能完全位于一个瓦片内部，因此，我们还需要计算坐标所在的瓦片和传入瓦片的差值，以及坐标所在瓦片的像素坐标，才能绘制出正确位置的要素。

```javascript
let lng = feature.geometry.coordinates[0],
    lat = feature.geometry.coordinates[1];
let tileXY = convert.lngLat2Tile(lng, lat, z);
let pixelXY = convert.lngLat2Pixel(lng, lat, z);
let pixelX = pixelXY.pixelX + (tileXY.tileX - x) * 256;
let pixelY = pixelXY.pixelY + (tileXY.tileY - y) * 256;
```

### convert

提供了 3 个方法（web 墨卡托投影）

1. lngLat2Tile 经纬度转换为瓦片号，非常常用
2. lngLat2Pixel 经纬度转换为瓦片号内的像素位置，后端渲染要素时需要用到，来绘制瓦片内的点线面
3. wgs84togcj02 代码中未使用

lnglat2Tile 函数是基于 web 墨卡托投影公式进行转换得到的，具体推导公式见[https://zhuanlan.zhihu.com/p/326955505](https://zhuanlan.zhihu.com/p/326955505)

### tile

提供了一个方法calc，返回具有空间索引(zooms)属性的feature

原理很简单，就是求出要素的最小外接矩形（xmin,ymin,xmax,ymax），然后调用convert.lngLat2Tile，就得到了tileMax、tileMin对应的tileX和tileY。同时也存储了pixelMin和pixelMax。但我们知道，在schema的索引里，并没有用到pixelMin和pixelMax，并且代码中其余位置也未使用，因此推断可能在前端用到。

其中，对于点（point）有些特殊，作者设定了一个buffer，用于计算点的最小外接矩形（x-buffer,y-buffer,x+buffer,y+buffer）。

### schema

全局变量 models 和 classes，分别存储 feature-class 的 name 对应的 model 与 document，会在更新数据库时更新这两个全局变量的值，读取直接读内存，不走数据库

这边的建立索引值得注意，

```javascript
const { minZoom = 0, maxZoom = 20 } = config.tile || {};
// add index, very important!!!
for (z = minZoom; z <= maxZoom; z++) {
  const index = {};
  index["zooms." + z + ".tileMin.tileX"] = 1;
  index["zooms." + z + ".tileMin.tileY"] = 1;
  index["zooms." + z + ".tileMax.tileX"] = -1;
  index["zooms." + z + ".tileMax.tileY"] = -1;
  schema.index(index, { name: "zoom_index_" + z });
}
models[cls.name] = mongoose.model(cls.name, schema);
```

可以看到这个 index 的结构是一个对象，格式为`zooms.16.tileMin.tileX`，这其实用到了 MongoDB 里的[Multikey indexes](https://www.mongodb.com/docs/manual/indexes/#multikey-index)

因为 tile 的结构如下：

```javascript
{
  zooms:[ // 数组索引表示对应的瓦片等级，范围从0-24
    {
      tileMin: {
        tileX: , // 横向瓦片号
        tileY: , // 纵向瓦片号
      },
      tileMax: {
        tileX: ,
        tileY: ,
      }
    }
  ]
}
```

表示先以 tileMin.tileX 和 tileMin.tileY 排序，然后再根据 tileMax.tileX 和 tileMax.tileY 排序

## routes

后端路由

### feature-classes

讲讲发布shapefile

1. 通过gdal打开shapefile，获取layers[0]，可以拿到geomType，和fields
2. 创建FeatureClass，加入schema，获取model
3. 遍历layer.features，获取geometry，坐标转为wgs84，调用tile.calc计算空间索引
4. 将features插入model

### features

1. 以geojson导出featureClass
从model中获取所有features，按FeatureCollection的格式写成geojson返回

2. 要素查询
支持查询要素的geometry、properties中的任意属性，拼接为查询范围在model中查询

### labels、maps、layers、symbols

支持创建、删除、更新、查询

### tiles

1. 获取用于前端渲染的矢量切片

查询路由：/vector/:name/:x/:y/:z

x、y、z都转为整数，组装成索引，在model中找出包含该xyz的瓦片

2. 获取后端渲染好的栅格图片

查询路由：/image/:id/:x/:y/:z

id表示class name或者layer id, 查询得到layer对象

调用canvas.draw(layer, x, y, z)，调用node-canvas的createPNGStream().pipe(res)，返回图片流

3. 获取静态切片图

查询路由：/static/:name/:x/:y/:z

直接组装为文件路径，返回图片文件

## 总结

1. 选mongodb做空间数据的数据库，对于矢量要素，空间索引自己做，效率可能会差一些，好处是实现简单
2. 在渲染侧除了提供常见的矢量要素，还提供了后端渲染矢量瓦片的功能，比较特别
3. 渲染样式根据openlayer的样式来自定义，同时可以按属性定值、条件渲染，有一些定制化的过滤方法
