---
title: "ElasticSearch官网翻译_GeoDistanceRangeQuery"
date: 2017-07-12 10:23:09
tags: [Elasticsearch]
categories: [Search]
---

## Geo Distance Range Query

[5.0.0] 5.0.0中弃用。 Elasticsearch将继续支持在5.0.0之前创建的索引的geo_distance_range查询。 将不再支持在5.0.0及更高版本中创建的索引的Range Query。 应使用距离聚合或排序。

过滤某一特定范围内存在的文档：

```
GET /_search
{
    "query": {
        "bool" : {
            "must" : {
                "match_all" : {}
            },
            "filter" : {
                "geo_distance_range" : {
                    "from" : "200km",
                    "to" : "400km",
                    "pin.location" : {
                        "lat" : 40,
                        "lon" : -70
                    }
                }
            }
        }
    }
}
```

支持与geo_distance过滤器相同的点位置参数和查询选项。 并且还支持范围（lt，lte，gt，gte，from，to，include_upper和include_lower）的常用参数。

### Ignore Unmapped（忽略未映射的）

当设置为true时，ignore_unmapped选项将忽略未映射字段，并且将不匹配此查询的任何文档。 当对可能具有不同映射的多个索引进行查询时，这可能很有用。 当设置为false（默认值）时，如果字段未映射，则查询将抛出异常。

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/5.4/query-dsl-geo-distance-range-query.html
