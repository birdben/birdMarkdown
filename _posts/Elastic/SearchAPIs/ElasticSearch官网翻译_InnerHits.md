---
title: "ElasticSearch官网翻译_InnerHits"
date: 2017-05-26 10:11:19
tags: [Elasticsearch]
categories: [Search]
---

#### Inner hits（内部命中）

<b>
Parent/Child和nested嵌套功能允许返回具有不同范围的匹配的文档。 在parent/child案例中，parent文档根据child文档中的匹配返回，或者根据parent文档中的匹配返回child文档。 在nested嵌套情况下，基于nested嵌套内部对象中的匹配返回文档。

在这两种情况下，导致在不同作用域中的实际匹配返回文档的被隐藏了。 在许多情况下，知道哪些内嵌嵌套对象（在nested嵌套情况下）或children/parent文档（在parent/child的情况下）导致某些信息被返回是非常有用的。 Inner hits功能可用于此。 此功能会在搜索响应中返回每个搜索匹配的附加nested嵌套命中，导致搜索匹配在不同的范围内匹配。
</b>

可以在nested，has_child或has_parent查询和过滤器之上定义inner_hits属性来使用inner hits。 结构如下所示：

```
"<query>" : {
    "inner_hits" : {
        <inner_hits_options>
    }
}
```

如果在支持它的查询上定义inner_hits，那么每个搜索命中将包含一个inner_hits json对象，具有以下结构：

```
"hits": [
     {
        "_index": ...,
        "_type": ...,
        "_id": ...,
        "inner_hits": {
           "<inner_hits_name>": {
              "hits": {
                 "total": ...,
                 "hits": [
                    {
                       "_type": ...,
                       "_id": ...,
                       ...
                    },
                    ...
                 ]
              }
           }
        },
        ...
     },
     ...
]
```

##### 选项

Inner hits支持以下选项：

值|描述
---|---
from|在返回的常规搜索匹配中，每个inner_hits的第一个命中提取的偏移量。
size|每个inner_hits返回的最大匹配次数。 默认情况下，返回前3个匹配的命中。
sort|内部命中应该如何根据inner_hits进行排序。 默认情况下，命中结果按score分数排序。
name|用于响应中特定inner hit内部命中定义的名称。 在单个搜索请求中定义了多个inner hit内部命中时很有用。 默认值取决于定义inner hit内部命中的查询。 对于has_child查询和过滤器是子类型，has_parent查询和过滤器是父类型和nested嵌套查询并过滤是nested path嵌套路径。

Inner hits还支持以下每个文档功能：

- Highlighting
- Explain
- Source filtering
- Script fields
- Fielddata fields
- Include versions

##### Nested inner hits（嵌套内部命中）

Nested的inner_hits可以用于将nested的内部对象作为搜索命中的内部命中。

下面的示例假定有一个嵌套对象字段定义并且名字为comments：

```
{
    "query" : {
        "nested" : {
            "path" : "comments",
            "query" : {
                "match" : {"comments.message" : "[actual query]"}
            },
            "inner_hits" : {} (1)
        }
    }
}
```

- (1) Inner hit定义在嵌套查询中。 没有其他选项需要定义。

可以从上述搜索请求生成的响应代码段的示例：

```
...
"hits": {
  ...
  "hits": [
     {
        "_index": "my-index",
        "_type": "question",
        "_id": "1",
        "_source": ...,
        "inner_hits": {
           "comments": { (1)
              "hits": {
                 "total": ...,
                 "hits": [
                    {
                       "_type": "question",
                       "_id": "1",
                       "_nested": {
                          "field": "comments",
                          "offset": 2
                       },
                       "_source": ...
                    },
                    ...
                 ]
              }
           }
        }
     },
     ...
```

- (1) 在搜索请求中的inner hit定义中使用的name。 可以通过name选项使用自定义key。

在上述示例中，_nested元数据至关重要，因为它定义了inner hit来自哪个内部嵌套对象。 field定义了嵌套命中来自的对象数组字段和相对于其在_source中的位置的offset。 由于排序和计分，inner_hits中的命中对象的实际位置通常不同于定义嵌套内部对象的位置。

默认情况下，还会为inner_hits中的命中对象返回_source，但这可以更改。 通过_source过滤功能，可以返回或禁用source源的一部分。 如果存储字段在嵌套级别上定义，那么也可以通过fields功能返回。

一个重要的缺省是在inner_hits内的命中中返回的_source是相对于_nested元数据的。 所以在上面的例子中，只有每个嵌套命中返回comment部分，而不是包含comment的顶级文档的整个source。

###### 注意

一个Elasticsearch 2.x的bug意味着如果你明确指定要作为inner_hits的_source的一部分返回的字段，则需要使用相对路径来定义它们，因此在上面的示例中必须写出：

```
"inner_hits" : {
     "_source":["message"]
     }
```

如果使用fielddata_fields返回字段数据，则需要指定完整路径。

##### Hierarchical levels of nested object fields and inner hits（嵌套对象字段和内部命中的层次级别）

如果映射具有多层次的层次化嵌套对象字段，则可以使用顶级内部命中访问每个级别（见下文）。

##### Parent/child inner hits

Parent/child inner_hits可用于包含parent或child

以下示例假定在comment类型中有一个_parent字段映射：

```
{
    "query" : {
        "has_child" : {
            "type" : "comment",
            "query" : {
                "match" : {"message" : "[actual query]"}
            },
            "inner_hits" : {} (1)
        }
    }
}
```

- (1) 内部命中定义就像在嵌套示例中一样。

可以从上述搜索请求生成的响应代码段的示例：

```
...
"hits": {
  ...
  "hits": [
     {
        "_index": "my-index",
        "_type": "question",
        "_id": "1",
        "_source": ...,
        "inner_hits": {
           "comment": {
              "hits": {
                 "total": ...,
                 "hits": [
                    {
                       "_type": "comment",
                       "_id": "5",
                       "_source": ...
                    },
                    ...
                 ]
              }
           }
        }
     },
     ...
```

##### Top level inner hits

除了在查询和过滤器上定义inner hits外，还可以将inner hits定义为query查询和aggregations聚合定义一起的顶级构造。 使用顶级inner hits定义的主要原因是让inner hits返回与主查询不匹配的文档。 Inner hits定义也可以通过顶级符号嵌套。 除此之外，查询中的inner hits定义应该被使用，因为这是定义inner hits的最紧凑的方式。

以下代码段解释了搜索请求正文顶层定义的inner hits的基本结构：

```
"inner_hits" : {
    "<inner_hits_name>" : {
        "<path|type>" : {
            "<path-to-nested-object-field|child-or-parent-type>" : {
                <inner_hits_body>
                [,"inner_hits" : { [<sub_inner_hits>]+ } ]?
            }
        }
    }
    [,"<inner_hits_name_2>" : { ... } ]*
}
```

在inner_hits定义中，首先定义inner hits的名称，然后定义inner_hit是否通过定义path或基于parent/child的定义来定义type来嵌套。 如果inner_hits是嵌套的，则下一个对象层包含嵌套对象字段的名称，如果inner_hit定义是基于parent/child的，则包含父或子类型。

可以在单个请求中定义多个inner hit定义。 在<inner_hits_body>中，可以定义inner_hits支持的功能的任何选项。 可选地，另一个inner_hits定义可以在<inner_hits_body>中定义。

一个示例，显示了通过顶级符号使用嵌套的inner hits：

```
{
    "query" : {
        "nested" : {
            "path" : "comments",
            "query" : {
                "match" : {"comments.message" : "[actual query]"}
            }
        }
    },
    "inner_hits" : {
        "comment" : {
            "path" : { (1)
                "comments" : { (2)
                    "query" : {
                        "match" : {"comments.message" : "[different query]"} (3)
                    }
                }
            }
        }
    }
}
```

- (1) inner hit定义是嵌套的，需要path选项。
- (2) path选项指的是嵌套对象字段comments
- (3) 运行的一个查询，用于为每个搜索命中收集嵌套的内部文档。 如果没有定义任何查询，所有嵌套的内部文档将被包含在搜索匹配中。 这表明，如果没有指定查询或者不同的查询，它只对顶级内部命中定义有意义。

仅在使用顶级inner hits符号时可用的附加选项：

值|描述
---|---
path|定义将从中收集匹配的嵌套范围。
type|定义要从中收集命中的parent或child类型分数。
query|定义将在定义的嵌套，parent或child范围中运行的查询，以收集和评分匹配。 默认情况下，范围内的所有文档将被匹配。

必须定义path或type。 path或type定义了从哪里获取并将其用作inner hits的范围。

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/2.4/search-request-inner-hits.html

