---
title: "ElasticSearch官网翻译_geohashPrefix"
date: 2017-05-18 18:58:07
tags: [Elasticsearch]
categories: [Search]
---

#### geohash_prefix

Geohash是一种将地球划分为网格的纬度/经度编码的形式。此网格中的每个单元格都由一个geohash字符串表示。 每个单元又可以进一步细分为更长的字符串表示的较小单元格。 因此，geohash（地理空间）越长，单元格越小（因此越准确）。

当geohash选项可以按照指定的精度对与lat/lon点对应的geohash进行索引时，geohash_prefix选项也会对所有包围的单元格进行索引。

例如，drm3btev3e86的geohash将会对所有以下terms进行索引：[d，dr，drm，drm3，drm3b，drm3bt，drm3bte，drm3btev，drm3btev3，drm3btev3e，drm3btev3e8，drm3btev3e86]。

geohash前缀可以与geohash_cell query一起使用，以查找特定geohash或其邻居点：

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
    "geohash_cell": {
      "location": {
        "lat": 41.02,
        "lon": -71.48
      },
      "precision": 4, (1)
      "neighbors": true (2)
    }
  }
}
```

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/2.4/geohash-prefix.html
