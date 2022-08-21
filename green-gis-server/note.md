# green-gis-server
[github](https://github.com/shengzheng1981/green-gis-server)

## 选择原因

如果有关注知乎webgis相关栏目的伙伴应该看过shengzheng的视频，就是介绍他的green-gis系列的课程，我看了他的代码仓库，也都是围绕green-gis拆分出来的几个部分。这个仓库是其中之一，看仓库名就了解，是一个后台服务系统。因为我还没毕业的时候就关注过这位大佬，但一直没有时间去看他的源码，现在工作了抽一些周末时间来补课。如果你对gis的原理感兴趣，也想自己搭建一个gis系统，不妨去知乎、b站来学习下这个系列的教程。

## 整体浏览

首先我看了一下仓库的目录，非常简单，public和views可以忽略，核心的文件夹就4个，bin、core、models、routes。

我又看了下README.md，features介绍了四个特性，

1. 发布shapefile
2. 返回feature collection的geojson
3. 如果是后端渲染，利用node-canvas返回动态的图片瓦片
4. 如果是前端渲染，根据瓦片的x/y/z返回feature colleciton

核心依赖：

1. mongodb + mongoose
2. node-gdal
3. node-canvas

再看下package.json，也非常简单，只有一个start命令和部分依赖，整个项目非常精简。

## bin/www

这个文件是启动文件，从package.json中可以看出，npm run start 执行的是 node ./bin/www

这个文件做了几件事：

1. 连接数据库mongodb
2. 创建http服务，监听端口
3. 创建了websocket，监听stream事件，并回传数据

亮点在于使用了mongoose，我查了一下，mongoose是mongodb的一个对象模型工具，封装了对Document的操作，更加便于使用。它有几个概念：

1. schema是对表结构的定义
2. model是由schema生成的模型，具有数据库操作的行为
3. entity是由model创建的实体，通过save方法保存数据

可以看到，在models文件夹下，其实就是定义了不同的schema和对应的model，这样很方便就能了解不同表的数据结构。

此外，在数据传输方面采用的websocket也是一个亮点。其中有一些小的技巧，例如，在features达到指定的buffer后才进行服务端->客户端的数据传输。这样的好处是减少传输的次数。

## models

## core

## routes

