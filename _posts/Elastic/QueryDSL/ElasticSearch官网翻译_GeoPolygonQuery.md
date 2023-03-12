---
title: "ElasticSearch官网翻译_GeoPolygonQuery"
date: 2017-07-12 10:24:06
tags: [Elasticsearch]
categories: [Search]
---

## Geo Polygon Query（多边形查询）

一个查询允许包含命中仅在多边形内的点。 这是一个例子：

```
GET /_search
{
    "query": {
        "bool" : {
            "must" : {
                "match_all" : {}
            },
            "filter" : {
                "geo_polygon" : {
                    "person.location" : {
                        "points" : [
                        {"lat" : 40, "lon" : -70},
                        {"lat" : 30, "lon" : -80},
                        {"lat" : 20, "lon" : -90}
                        ]
                    }
                }
            }
        }
    }
}
```

#### Query Options（查询选项）

选项|描述
---|---
_name|可选名称字段来标识过滤器
ignore_malformed|[5.0.0] 5.0.0中弃用。 使用validation_method替代。 设置为true以接受无效的latitude（纬度）或longitude（经度）的geo points（地理点）（默认为false）。
validation_method|设置为IGNORE_MALFORMED以接受无效latitude（纬度）或longitude（经度）的geo points（地理点），设置为COERCE也可以尝试推断正确的latitude（纬度）或longitude（经度）。 （默认为STRICT）。

#### Accepted Formats（接受的格式）

##### Lat Long as Array

格式为[lon, lat]，注意，这里的lon/lat的顺序是为了符合GeoJSON。

```
GET /_search
{
    "query": {
        "bool" : {
            "must" : {
                "match_all" : {}
            },
            "filter" : {
                "geo_polygon" : {
                    "person.location" : {
                        "points" : [
                            [-70, 40],
                            [-80, 30],
                            [-90, 20]
                        ]
                    }
                }
            }
        }
    }
}
```

##### Lat Lon As String

格式为lat,lon。

```
GET /_search
{
    "query": {
        "bool" : {
            "must" : {
                "match_all" : {}
            },
            "filter" : {
               "geo_polygon" : {
                    "person.location" : {
                        "points" : [
                            "40, -70",
                            "30, -80",
                            "20, -90"
                        ]
                    }
                }
            }
        }
    }
}
```

##### Geohash

```
GET /_search
{
    "query": {
        "bool" : {
            "must" : {
                "match_all" : {}
            },
            "filter" : {
               "geo_polygon" : {
                    "person.location" : {
                        "points" : [
                            "drn5x1g8cu2y",
                            "30, -80",
                            "20, -90"
                        ]
                    }
                }
            }
        }
    }
}
```

#### geo_point Type（地理点类型）

该查询需要在相关字段上设置geo_point类型。

#### Ignore Unmapped（忽略未映射的）

当设置为true时，ignore_unmapped选项将忽略未映射字段，并且将不匹配此查询的任何文档。 当对可能具有不同映射的多个索引进行查询时，这可能很有用。 当设置为false（默认值）时，如果字段未映射，则查询将抛出异常。

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/5.4/query-dsl-geo-polygon-query.html
