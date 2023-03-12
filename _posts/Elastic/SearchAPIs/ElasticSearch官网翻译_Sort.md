---
title: "ElasticSearch官网翻译_Sort"
date: 2017-05-24 14:53:16
tags: [Elasticsearch]
categories: [Search]
---

#### Sort

允许在特定字段上添加一个或多个排序。 每种排序也可以反转。 排序在每个字段级别定义，_score的特殊字段名称按照score得分排序，_doc按索引顺序排序。

```
{
    "sort" : [
        { "post_date" : {"order" : "asc"}},
        "user",
        { "name" : "desc" },
        { "age" : "desc" },
        "_score"
    ],
    "query" : {
        "term" : { "user" : "kimchy" }
    }
}
```

###### 注意

<b>除了最有效的排序顺序之外，_doc还没有真正的用例。 所以如果你不关心文档的返回顺序，那么你应该按_doc排序。 这在scrolling时尤其有帮助。</b>

##### Sort Values

返回的每个文档的排序值也作为响应的一部分返回。

##### Sort Order

order选项可以具有以下值：

值|描述
---|---
asc|按升序排列
desc|按降序排列

<b>按照_score排序时，order默认为desc，并且在其他任何内容排序时默认为asc。</b>

##### 排序模式选项

Elasticsearch支持通过数组或多值字段进行排序。 mode选项控制选择哪个数组值来排序它所属的文档。 mode选项可以具有以下值：

值|描述
---|---
min|选择最小值。
max|选择最大值。
sum|使用所有值的和作为排序值。 仅适用于基于数字的数组字段。
avg|使用所有值的平均值作为排序值。 仅适用于基于数字的数组字段。
median|使用所有值的中间数作为排序值。 仅适用于基于数字的数组字段。

排序模式示例用法

在下面的示例中，字段price每个文档有多个价格。 在这种情况下，命中结果将根据每个文档的平均价格按升序排序。

```
curl -XPOST 'localhost:9200/_search' -d '{
   "query" : {
    ...
   },
   "sort" : [
      {"price" : {"order" : "asc", "mode" : "avg"}}
   ]
}'
```

##### 在嵌套对象中排序

Elasticsearch还支持按一个或多个嵌套对象内的字段进行排序。 通过嵌套字段支持的排序在已经存在的排序选项之上具有以下参数：

值|描述
---|---
nested_path|<b>定义要排序的嵌套对象。 实际排序字段必须是此嵌套对象内的直接字段。 当通过嵌套字段排序时，此字段是必需的。</b>
nested_filter|<b>嵌套路径中的内部对象应该匹配的过滤器，以便通过排序来考虑其字段值。 常见的情况是重复嵌套过滤器或查询中的查询/过滤器。 默认情况下没有nested_filter是active的。</b>

嵌套排序示例

在下面的示例中，offer是一个nested嵌套类型的字段。 需要指定nested_path; 否则，Elasticsearch不知道需要捕获什么嵌套级排序值。

```
curl -XPOST 'localhost:9200/_search' -d '{
   "query" : {
    ...
   },
   "sort" : [
       {
          "offer.price" : {
             "mode" :  "avg",
             "order" : "asc",
             "nested_path" : "offer",
             "nested_filter" : {
                "term" : { "offer.color" : "blue" }
             }
          }
       }
    ]
}'
```

通过scripts脚本排序和按geo distance距离排序也支持嵌套排序。

##### 缺少值

<b>missing参数指定如何处理缺少字段的文档：missing的值可以设置为_last，_first或自定义值（将用作缺少的文档作为排序值）。 </b>例如：

```
{
    "sort" : [
        { "price" : {"missing" : "_last"} },
    ],
    "query" : {
        "term" : { "user" : "kimchy" }
    }
}
```

###### 注意

<b>如果嵌套的内部对象与nested_filter不匹配，则使用缺少的值。</b>

##### 忽略未映射的字段

默认情况下，如果没有与字段关联的映射，则搜索请求将失败。 unmapped_type选项允许忽略没有映射的字段，而不对它们进行排序。 此参数的值用于确定要返回的排序值。 这是一个如何使用它的例子：

```
{
    "sort" : [
        { "price" : {"unmapped_type" : "long"} },
    ],
    "query" : {
        "term" : { "user" : "kimchy" }
    }
}
```

<b>如果查询中的任何一个索引都没有price的映射，那么Elasticsearch将处理它，就好像有一个类型为long的映射，该索引中的所有文档都没有该字段的值。</b>

##### Geo Distance Sorting

允许通过_geo_distance进行排序。 这是一个例子：

```
{
    "sort" : [
        {
            "_geo_distance" : {
                "pin.location" : [-70, 40],
                "order" : "asc",
                "unit" : "km",
                "mode" : "min",
                "distance_type" : "sloppy_arc"
            }
        }
    ],
    "query" : {
        "term" : { "user" : "kimchy" }
    }
}
```

值|描述
---|---
distance_type|如何计算距离。 可以是sloppy_arc（默认），arc（稍微更精确但明显更慢）或plane（更快，但长距离不准确，靠近极点）。

注意：geo distance sorting支持sort_mode选项：min，max和avg。

提供以下格式支持坐标：

Lat Lon as Properties

```
{
    "sort" : [
        {
            "_geo_distance" : {
                "pin.location" : {
                    "lat" : 40,
                    "lon" : -70
                },
                "order" : "asc",
                "unit" : "km"
            }
        }
    ],
    "query" : {
        "term" : { "user" : "kimchy" }
    }
}
```

Lat Lon as String

格式为lat, lon。

```
{
    "sort" : [
        {
            "_geo_distance" : {
                "pin.location" : "40,-70",
                "order" : "asc",
                "unit" : "km"
            }
        }
    ],
    "query" : {
        "term" : { "user" : "kimchy" }
    }
}
```

Geohash

```
{
    "sort" : [
        {
            "_geo_distance" : {
                "pin.location" : "drm3btev3e86",
                "order" : "asc",
                "unit" : "km"
            }
        }
    ],
    "query" : {
        "term" : { "user" : "kimchy" }
    }
}
```

Lat Lon as Array

格式为[lon, lat]，注意，这里的lon/lat的顺序是为了符合GeoJSON。

```
{
    "sort" : [
        {
            "_geo_distance" : {
                "pin.location" : [-70, 40],
                "order" : "asc",
                "unit" : "km"
            }
        }
    ],
    "query" : {
        "term" : { "user" : "kimchy" }
    }
}
```

##### 多个参考点

例如，多个地理点可以作为包含任何geo_point格式的数组传递

```
"pin.location" : [[-70, 40], [-71, 42]]
"pin.location" : [{"lat": 40, "lon": -70}, {"lat": 42, "lon": -71}]
```

等等。

<b>文档的最终距离将会是文档中包含的所有点的min/max/avg（通过mode定义）到排序请求中给出的所有点。</b>

##### 基于脚本的排序

允许根据自定义脚本进行排序，这里是一个例子：

```
{
    "query" : {
        ....
    },
    "sort" : {
        "_script" : {
            "type" : "number",
            "script" : {
                "inline": "doc['field_name'].value * factor",
                "params" : {
                    "factor" : 1.1
                }
            },
            "order" : "asc"
        }
    }
}
```

##### 追踪得分

在字段排序时，不计算分数。 通过将track_scores设置为true，仍将计算和跟踪分数。

```
{
    "track_scores": true,
    "sort" : [
        { "post_date" : {"order" : "desc"} },
        { "name" : "desc" },
        { "age" : "desc" }
    ],
    "query" : {
        "term" : { "user" : "kimchy" }
    }
}
```

##### 内存注意事项

排序时，相关的排序字段值将加载到内存中。 这意味着每个分片应该有足够的内存来容纳它们。 对于基于字符串的类型，排序的字段不应该被analyzed（分析）/tokenized（标记化）。 对于数字类型，如果可以，建议将类型显式设置为较窄类型（如short，integer和float）。

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/2.4/search-request-sort.html

