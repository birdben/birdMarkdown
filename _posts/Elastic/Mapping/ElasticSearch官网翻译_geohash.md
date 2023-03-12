---
title: "ElasticSearch官网翻译_geohash"
date: 2017-05-13 18:55:14
tags: [Elasticsearch]
categories: [Search]
---

#### geohash（地理位置）

Geohash是一种将地球划分为网格的纬度/经度编码的形式。 此网格中的每个单元格都由一个geohash字符串表示。 每个单元又可以进一步细分为更长的字符串表示的较小单元格。 因此，geohash（地理空间）越长，单元格越小（因此越准确）。

因为geohash只是字符串，它们可以像任何其他字符串一样存储在倒排索引中，这使得查询非常高效。

如果启用了geohash选项，则geohash "sub-field"（子字段）将被索引为例如：.geohash。 geohash的长度由geohash_precision参数控制。

如果启用了geohash_prefix选项，则会自动启用geohash选项。

举例：

```
PUT my_index
{
  "mappings": {
    "my_type": {
      "properties": {
        "location": {
          "type": "geo_point", (1)
          "geohash": true
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

GET my_index/_search?fielddata_fields=location.geohash (2)
{
  "query": {
    "prefix": {
      "location.geohash": "drm3b" (3)
    }
  }
}
```

- (1) 将为每个geo-point地理位置索引为location.geohash字段。
- (2) 可以用doc_values检索geohash。
- (3) prefix query前缀查询可以找到以特定前缀开头的所有geohashes地理位置。

###### 警告

对地理位置的前缀查询是昂贵的。 相反，请考虑使用geohash_prefix在索引时付出代价，而不是在每个查询上付出代价。

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/2.4/geohash.html
