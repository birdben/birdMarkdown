---
title: "ElasticSearch官网翻译_GeoPointDatatype"
date: 2017-05-11 15:13:36
tags: [Elasticsearch]
categories: [Search]
---

#### Geo-point datatype

geo_point类型的字段可以接受"纬度-经度"对，可以使用：

- 在一个边界框内，在一个中心点的一定距离内，一个多边形内或在一个geohash单元格内找到地理点。
- 通过地理位置或距离中心点的距离来汇总文档。
- 将距离整合到文档的相关性分数中。
- 按距离排序文档。

##### 重要 : 在Elasticsearch 2.2.0或更高版本中Percolating geo-queries（渗透地理查询）

<b>在Elasticsearch 2.2.0及更高版本中添加的新geo_point字段要求启用doc_value才能运行。 不幸的是，由percolator使用的内存中索引尚未支持doc_value，这意味着geo-queries（地理查询）将无法在Elasticsearch 2.2.0或更高版本中创建的percolator（渗透）索引中使用。</b>

可以指定geo-point地理位置的四种方式如下所示：

```
PUT my_index
{
  "mappings": {
    "my_type": {
      "properties": {
        "location": {
          "type": "geo_point"
        }
      }
    }
  }
}

PUT my_index/my_type/1
{
  "text": "Geo-point as an object",
  "location": { (1)
    "lat": 41.12,
    "lon": -71.34
  }
}

PUT my_index/my_type/2
{
  "text": "Geo-point as a string",
  "location": "41.12,-71.34" (2)
}

PUT my_index/my_type/3
{
  "text": "Geo-point as a geohash",
  "location": "drm3btev3e86" (3)
}

PUT my_index/my_type/4
{
  "text": "Geo-point as an array",
  "location": [ -71.34, 41.12 ] (4)
}

GET my_index/_search
{
  "query": {
    "geo_bounding_box": { (5)
      "location": {
        "top_left": {
          "lat": 42,
          "lon": -72
        },
        "bottom_right": {
          "lat": 40,
          "lon": -74
        }
      }
    }
  }
}
```

- (1) Geo-point表示为一个对象，具有lat和lon关键字。
- (2) Geo-point表示为格式为"lat，lon"的字符串。
- (3) Geo-point表示为geohash。
- (4) Geo-point表示为数组，格式为：[lon，lat]。
- (5) 查找位于框内的所有geo-points的地理边界框查询。

##### 重要 : Geo-points表示为数组或字符串

<b>
请注意，字符串Geo-points按照lat，lon排序，而数组Geo-point按照相反的顺序排列：lon，lat。

最初，lat，lon都用于数组和字符串，但是数组格式早期被更改为符合GeoJSON使用的格式。
</b>


#### geo_point字段的参数

geo_point字段接受以下参数：

- geohash : geo-point也应该在.geohash子字段中作为geohash进行索引？ 默认为false，除非geohash_prefix为true。 [2.4中弃用]。

- geohash_precision : 用于geohash和geohash_prefix选项的geohash的最大长度。 [2.4中弃用]。

- geohash_prefix : geo-point还应该作为geohash加上所有的前缀进行索引？ 默认为false。 [2.4中弃用]。

- ignore_malformed : 如果为true，则会忽略格式不正确的geo-point。 如果为false（默认），格式错误的geo-point会抛出异常并拒绝整个文档。

- lat_lon : 地理位置是否也被索引为.lat和.lon子字段？ 接受true和false（默认）。[2.3中弃用]

#### 在脚本中使用地理位置

当访问脚本中的geo-point值时，该值将返回为GeoPoint对象，该对象允许分别访问.lat和.lon值：

```
geopoint = doc['location'].value;
lat      = geopoint.lat;
lon      = geopoint.lon;
```

出于性能原因，最好直接访问lat/lon的值：

```
lat      = doc['location'].lat;
lon      = doc['location'].lon;
```

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/2.4/geo-point.html