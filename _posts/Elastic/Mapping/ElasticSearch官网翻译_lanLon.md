---
title: "ElasticSearch官网翻译_latLon"
date: 2017-05-19 20:16:56
tags: [Elasticsearch]
categories: [Search]
---

#### lat_lon

<b>
Geo-queries（地理查询）通常通过将每个geo_point字段的值插入公式来执行，以确定它是否落入所需的区域。 与大多数查询不同，不需要倒排索引。

将lat_lon设置为true会使纬度和经度值作为数字字段（称为.lat和.lon）进行索引。 geo_bounding_box和geo_distance查询可以使用这些字段，而不是执行内存计算。
</b>

```
PUT my_index
{
  "mappings": {
    "my_type": {
      "properties": {
        "location": {
          "type": "geo_point",
          "lat_lon": true 
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


GET my_index/_search
{
  "query": {
    "geo_distance": {
      "location": {
        "lat": 41,
        "lon": -71
      },
      "distance": "50km",
      "optimize_bbox": "indexed" 
    }
  }
}
```

- (1) 将lat_lon设置为true可以对location.lat和location.lon字段中的geo-point（地理位置）进行索引。
- (2) indexed选项指示geo-distance query（地理距离查询）使用倒排索引而不是内存中的计算。

内存计算或索引操作是否有更好地执行性能取决于你的数据集和正在运行的查询的类型。

###### 注意

lat_lon选项只对单值geo_point字段有意义。 它不适用于arrays of geo-points（地理位置数组）。

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/2.4/lat-lon.html

