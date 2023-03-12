---
title: "ElasticSearch官网翻译_GeoBoundingBoxQuery"
date: 2017-06-30 19:19:17
tags: [Elasticsearch]
categories: [Search]
---

## Geo Bounding Box Query

允许使用bounding box（边界框）根据点位置过滤命中的查询。 假设以下索引文档：

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

然后可以使用geo_bounding_box过滤器执行以下简单查询：

```
GET /_search
{
    "query": {
        "bool" : {
            "must" : {
                "match_all" : {}
            },
            "filter" : {
                "geo_bounding_box" : {
                    "pin.location" : {
                        "top_left" : {
                            "lat" : 40.73,
                            "lon" : -74.1
                        },
                        "bottom_right" : {
                            "lat" : 40.01,
                            "lon" : -71.12
                        }
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
ignore_malformed|5.0.0中弃用。使用validation_method替代。 设置为true以接受无效的latitude（纬度）或longitude（经度）的geo points（地理点）（默认为false）。
validation_method|设置为IGNORE_MALFORMED以接受无效latitude（纬度）或longitude（经度）的geo points（地理点），设置为COERCE也可以尝试推断正确的latitude（纬度）或longitude（经度）。 （默认为STRICT）。
type|设置为indexed或memory，以定义此过滤器将在内存中执行还是索引时执行。 有关详细信息，请参阅下面的Type。默认值是memory。

##### Accepted Formats（接受的格式）

以同样的方式，geo_point类型可以接受geo point（地理点）的不同表示，过滤器也可以接受它：

##### Lat Lon As Properties

```
GET /_search
{
    "query": {
        "bool" : {
            "must" : {
                "match_all" : {}
            },
            "filter" : {
                "geo_bounding_box" : {
                    "pin.location" : {
                        "top_left" : {
                            "lat" : 40.73,
                            "lon" : -74.1
                        },
                        "bottom_right" : {
                            "lat" : 40.01,
                            "lon" : -71.12
                        }
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
GET /_search
{
    "query": {
        "bool" : {
            "must" : {
                "match_all" : {}
            },
            "filter" : {
                "geo_bounding_box" : {
                    "pin.location" : {
                        "top_left" : [-74.1, 40.73],
                        "bottom_right" : [-71.12, 40.01]
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
                "geo_bounding_box" : {
                    "pin.location" : {
                        "top_left" : "40.73, -74.1",
                        "bottom_right" : "40.01, -71.12"
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
                "geo_bounding_box" : {
                    "pin.location" : {
                        "top_left" : "dr5r9ydj2y73",
                        "bottom_right" : "drj7teegpus6"
                    }
                }
            }
        }
    }
}
```

#### Vertices（顶点）

边框的顶点可以由top_left和bottom_right或者top_right和bottom_left参数设置。 此外topLeft，bottomRight，topRight和bottomLeft的名称也被支持。 为了替换成对设置值，可以使用简单的名称top，left，bottom和right单独设置值。

```
GET /_search
{
    "query": {
        "bool" : {
            "must" : {
                "match_all" : {}
            },
            "filter" : {
                "geo_bounding_box" : {
                    "pin.location" : {
                        "top" : 40.73,
                        "left" : -74.1,
                        "bottom" : 40.01,
                        "right" : -71.12
                    }
                }
            }
        }
    }
}
```

#### geo_point Type（地理点类型）

过滤器需要在相关字段上设置geo_point类型。

#### Multi Location Per Document（每个文档多个位置）

过滤器可以处理每个文档的多个locations/points。 一旦单个location/point与过滤器匹配，文档将被包含在过滤器中

#### Type（类型）

默认情况下，bounding box（边框）执行的类型设置为memory，这意味着在内存中检查文档是否在bounding box（边框）范围内。 在某些情况下，indexed选项的执行速度会更快（但在这种情况下，请注意geo_point类型必须已经被索引为lat和lon）。 请注意，使用indexed选项时，不支持每个文档字段多个位置。 这是一个例子：

```
GET /_search
{
    "query": {
        "bool" : {
            "must" : {
                "match_all" : {}
            },
            "filter" : {
                "geo_bounding_box" : {
                    "pin.location" : {
                        "top_left" : {
                            "lat" : 40.73,
                            "lon" : -74.1
                        },
                        "bottom_right" : {
                            "lat" : 40.10,
                            "lon" : -71.12
                        }
                    },
                    "type" : "indexed"
                }
            }
        }
    }
}
```

#### Ignore Unmapped（忽略未映射的）

当设置为true时，ignore_unmapped选项将忽略未映射字段，并且将不匹配此查询的任何文档。 当对可能具有不同映射的多个索引进行查询时，这可能很有用。 当设置为false（默认值）时，如果字段未映射，则查询将抛出异常。

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/5.4/query-dsl-geo-bounding-box-query.html
