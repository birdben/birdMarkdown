---
title: "ElasticSearch官网翻译_GeoDistanceQuery"
date: 2017-06-30 19:52:32
tags: [Elasticsearch]
categories: [Search]
---

## Geo Distance Query

过滤仅包含命中在地理位置特定距离内的文档。 假设以下映射和索引的文档：

```
PUT /my_locations
{
    "mappings": {
        "location": {
            "properties": {
                "pin": {
                    "properties": {
                        "location": {
                            "type": "geo_point"
                        }
                    }
                }
            }
        }
    }
}

PUT /my_locations/location/1
{
    "pin" : {
        "location" : {
            "lat" : 40.12,
            "lon" : -71.34
        }
    }
}
```

然后可以使用geo_distance过滤器执行以下简单查询：

```
GET /my_locations/location/_search
{
    "query": {
        "bool" : {
            "must" : {
                "match_all" : {}
            },
            "filter" : {
                "geo_distance" : {
                    "distance" : "200km",
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

#### Accepted Formats（接受的格式）

以同样的方式，geo_point类型可以接受geo point（地理点）的不同表示，过滤器也可以接受它：

##### Lat Lon As Properties

```
GET /my_locations/location/_search
{
    "query": {
        "bool" : {
            "must" : {
                "match_all" : {}
            },
            "filter" : {
                "geo_distance" : {
                    "distance" : "12km",
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

##### Lat Lon As Array

格式为[lon, lat]，注意，这里的lon/lat的顺序是为了符合GeoJSON。

```
GET /my_locations/location/_search
{
    "query": {
        "bool" : {
            "must" : {
                "match_all" : {}
            },
            "filter" : {
                "geo_distance" : {
                    "distance" : "12km",
                    "pin.location" : [-70, 40]
                }
            }
        }
    }
}
```

##### Lat Lon As String

格式为lat,lon。

```
GET /my_locations/location/_search
{
    "query": {
        "bool" : {
            "must" : {
                "match_all" : {}
            },
            "filter" : {
                "geo_distance" : {
                    "distance" : "12km",
                    "pin.location" : "40,-70"
                }
            }
        }
    }
}
```

##### Geohash

```
GET /my_locations/location/_search
{
    "query": {
        "bool" : {
            "must" : {
                "match_all" : {}
            },
            "filter" : {
                "geo_distance" : {
                    "distance" : "12km",
                    "pin.location" : "drm3btev3e86"
                }
            }
        }
    }
}
```

#### Options（选项）

以下是过滤器上允许的选项：

值|描述
---|---
distance|以指定位置为中心的圆的半径。 落入此圆形内的点被认为是匹配。 distance可以以各种单位指定。 请参阅"Distance Units"章节。
distance_type|如何计算距离。 可以是arc（弧）（默认值）或plane（平面）（更快，但长距离不准确，接近极点）。
optimize_bbox|是否在distance（距离）检查之前先使用bounding box（边框）检查进行优化。 默认值为memory，即在内存中进行检查。 也可以设置为indexed在索引时进行检查（但在这种情况下，请注意geo_point类型必须已经被索引为lat和lon），或者设置为none禁用边界框优化。 [2.2]版本中弃用
_name|可选名称字段来标识查询
ignore_malformed|[5.0.0]版本中弃用。 使用validation_method替代。设置为true以接受无效的latitude（纬度）或longitude（经度）的geo points（地理点）（默认为false）。
validation_method|设置为IGNORE_MALFORMED以接受无效latitude（纬度）或longitude（经度）的geo points（地理点），设置为COERCE也可以尝试推断正确的latitude（纬度）或longitude（经度）。 （默认为STRICT）。

#### geo_point Type（地理点类型）

过滤器需要在相关字段上设置geo_point类型。

#### Multi Location Per Document（每个文档多个位置）

geo_distance过滤器可以处理每个文档的多个locations/points。 一旦单个location/point与过滤器匹配，文档将被包含在过滤器中。

#### Ignore Unmapped（忽略未映射的）

当设置为true时，ignore_unmapped选项将忽略未映射字段，并且将不匹配此查询的任何文档。 当对可能具有不同映射的多个索引进行查询时，这可能很有用。 当设置为false（默认值）时，如果字段未映射，则查询将抛出异常。

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/5.4/query-dsl-geo-distance-query.html
