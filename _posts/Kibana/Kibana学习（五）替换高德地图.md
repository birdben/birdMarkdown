---
title: "Kibana学习（五）替换高德地图"
date: 2017-01-23 17:16:10
tags: [Kibana]
categories: [Log]
---

### 修改地图配置

找到src/ui/public/vislib/visualizations/_map.js文件，修改请求的地图API

##### 替换成高德地图

```
var mapTiles = {
  //url: tilemap.url,
  //options: _.assign({}, tilemapOptions, { attribution })
  // 替换成高德地图，参数：lang指定显示语言；style指定地图的风格，一般使用7
  url: 'http://webst0{s}.is.autonavi.com/appmaptile?lang=zh_cn&style=7&x={x}&y={y}&z={z}',
  options: {
    attribution: 'Tiles by <a href="http://www.mapquest.com/">MapQuest</a> &mdash; ' +
      'Map data &copy; <a href="http://openstreetmap.org">OpenStreetMap</a> contributors, ' +
      '<a href="http://creativecommons.org/licenses/by-sa/2.0/">CC-BY-SA</a>',
    subdomains: '1234'
  }
};
```

##### 修改地图的默认展示

```
//修改为５，默认展示地图，当值为２是默认展示空白
//var defaultMapZoom = 2;
var defaultMapZoom = 5;

//修改默认展示中国地图的位置
//var defaultMapCenter = [15, 5];
var defaultMapCenter = [32.97180377635759, 108.28125];
```

### 进行编译

K4有一套机制是要通过webpack来进行编译的，从而达到大数据多而不崩，畅而不卡的效果所架构的，及时修改源代码也要编译后才能用，所以要删除${KB_HOME}/optimize/bundles目录，然后重启Kibana重新进行编译。在这一步Kibana启动时间会比平时慢，往往会卡停一段时间，因为此时正在进行编译创造bundles文件夹编译完之后发现之前删除掉的bundles又出现了。

##### Kibana启动日志

```
{"type":"log","@timestamp":"2017-01-23T07:39:18+00:00","tags":["info","optimize"],"pid":12483,"message":"Optimizing and caching bundles for sense, kibana and statusPage. This may take a few minutes"}
{"type":"log","@timestamp":"2017-01-23T07:39:53+00:00","tags":["info","optimize"],"pid":12653,"message":"Optimizing and caching bundles for sense, kibana and statusPage. This may take a few minutes"}
{"type":"log","@timestamp":"2017-01-23T07:40:35+00:00","tags":["info","optimize"],"pid":12653,"message":"Optimization of bundles for sense, kibana and statusPage complete in 42.81 seconds"}
```


### 问题总结

- Kibana地图空白，没有地图形状

这个是默认显示的是空白，只要点放大的那个 + 号，就可以看到地图形状了。或者可以按照上面配置修改defaultMapZoom的值为5，就可以避免默认展示空白。

![Kibana地图空白](http://elasticsearch.cn/uploads/questions/20151229/3baaef1d482777536a4739eca06f371d.png)

- Kibana默认显示中国地图

如上面配置修改defaultMapCenter默认显示的位置即可

- 重启Kibana半天没有反应

修改后第一次启动的时候会比较慢，因为Kibana使用webpack进行编译的，从而提供高效稳定的性能。修改源码后需要编译后才能运行，上面的删除optimize/bundles就是删除编译的结果，让kibana重新编译


参考文章：

- http://elasticsearch.cn/question/281
- http://2002wmj.iteye.com/blog/2311042