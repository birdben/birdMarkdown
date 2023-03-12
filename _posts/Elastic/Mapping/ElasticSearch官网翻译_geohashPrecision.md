---
title: "ElasticSearch官网翻译_geohashPrecision"
date: 2017-05-18 17:18:53
tags: [Elasticsearch]
categories: [Search]
---

#### geohash_precision

Geohash是一种将地球划分为网格的纬度/经度编码的形式。此网格中的每个单元格都由一个geohash字符串表示。 每个单元又可以进一步细分为更长的字符串表示的较小单元格。 因此，geohash（地理空间）越长，单元格越小（因此越准确）。

geohash_precision设置控制在启用geohash选项时索引的geohash的长度，以及geohash_prefix选项启用时的最大geohash长度。

它接受：

- 1到12之间的数字（默认值），表示geohash的长度。
- 一段距离，例如：一公里。

如果指定了距离，它将被转换为将提供所请求的分辨率的最小geohash长度。

例如：

```
PUT my_index
{
  "mappings": {
    "my_type": {
      "properties": {
        "location": {
          "type": "geo_point",
          "geohash_prefix": true,
          "geohash_precision": 6 
        }
      }
    }
  }
}

PUT my_index/my_type/1
{
  "location": {
    "lat": 41.12,
    "lon": -71.34
  }
}

GET my_index/_search?fielddata_fields=location.geohash
{
  "query": {
    "term": {
      "location.geohash": "drm3bt"
    }
  }
}
```

- (1) geohash_precision为6代表等于大约1.26km×0.6km的geohash单元格


参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/2.4/geohash-precision.html
