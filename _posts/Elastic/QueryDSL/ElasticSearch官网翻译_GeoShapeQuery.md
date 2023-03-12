---
title: "ElasticSearch官网翻译_GeoShapeQuery"
date: 2017-06-30 18:41:35
tags: [Elasticsearch]
categories: [Search]
---

## GeoShape Query

过滤使用geo_shape类型索引的文档。

需要geo_shape Mapping。

geo_shape查询使用与geo_shape映射相同的网格正方形表示，来查找具有与query shape（查询形状）相交形状的文档。 它还将使用与字段映射定义相同的PrefixTree配置。

该查询支持两种方式定义查询形状：通过提供整体形状定义，或通过引用另一索引中pre-indexed（预索引）的形状的名称。 这两种格式都在下面的例子中定义。

#### Inline Shape Definition（内联形状定义）

与geo_shape类型类似，geo_shape查询使用GeoJSON来表示形状。

给出以下索引：

```
PUT /example
{
    "mappings": {
        "doc": {
            "properties": {
                "location": {
                    "type": "geo_shape"
                }
            }
        }
    }
}

POST /example/doc?refresh
{
    "name": "Wind & Wetter, Berlin, Germany",
    "location": {
        "type": "point",
        "coordinates": [13.400544, 52.530286]
    }
}
```

以下查询将使用Elasticsearch的envelope类型的GeoJSON扩展来找到该点：

```
GET /example/_search
{
    "query":{
        "bool": {
            "must": {
                "match_all": {}
            },
            "filter": {
                "geo_shape": {
                    "location": {
                        "shape": {
                            "type": "envelope",
                            "coordinates" : [[13.0, 53.0], [14.0, 52.0]]
                        },
                        "relation": "within"
                    }
                }
            }
        }
    }
}
```

#### Pre-Indexed Shape（预索引形状）

查询还支持使用已在另一个索引和（或）索引类型中已经被索引的形状。 这个特别有用，对于你有一个预定义的形状列表，这个预定义的形状列表对你的应用程序有用，并且你想使用逻辑名称（例如New Zealand）引用此形状而不是每次都提供其坐标。 在这种情况下，只需要提供：

- id - 包含预索引形状的文档的ID。
- index - 预索引形状的索引名称。 默认为shapes。
- type - 预索引形状的索引类型。
- path - 指定为包含预索引形状的路径的字段。 默认为shape。

以下是使用具有预索引形状的过滤器的示例：

```
PUT /shapes
{
    "mappings": {
        "doc": {
            "properties": {
                "location": {
                    "type": "geo_shape"
                }
            }
        }
    }
}

PUT /shapes/doc/deu
{
    "location": {
        "type": "envelope",
        "coordinates" : [[13.0, 53.0], [14.0, 52.0]]
    }
}

GET /example/_search
{
    "query": {
        "bool": {
            "filter": {
                "geo_shape": {
                    "location": {
                        "indexed_shape": {
                            "index": "shapes",
                            "type": "doc",
                            "id": "deu",
                            "path": "location"
                        }
                    }
                }
            }
        }
    }
}
```

#### Spatial Relations（空间关系）

geo_shape strategy（策略）映射参数确定在搜索时可以使用哪些空间关系运算符。

以下是可用的空间关系运算符的完整列表：

- INTERSECTS - （默认）返回geo_shape字段与查询几何相交的所有文档。
- DISJOINT - 返回geo_shape字段与查询几何相同的所有文档。
- WITHIN - 返回geo_shape字段在查询几何中的所有文档。
- CONTAINS - 返回geo_shape字段包含查询几何的所有文档。

#### Ignore Unmapped（忽略未映射的）

当设置为true时，ignore_unmapped选项将忽略未映射字段，并且将不匹配此查询的任何文档。 当对可能具有不同映射的多个索引进行查询时，这可能很有用。 当设置为false（默认值）时，如果字段未映射，则查询将抛出异常。

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/5.4/query-dsl-geo-shape-query.html
