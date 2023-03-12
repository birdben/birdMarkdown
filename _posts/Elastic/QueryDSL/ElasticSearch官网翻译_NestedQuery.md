---
title: "ElasticSearch官网翻译_NestedQuery"
date: 2017-06-30 16:14:05
tags: [Elasticsearch]
categories: [Search]
---

## Nested Query（嵌套查询）

Nested query允许查询嵌套objects/docs（参见nested mapping）。 该查询是针对嵌套的objects/docs执行的，就好像它们作为单独的文档（在内部他们是单独的文档）进行索引，返回的文档是根父文档（或父嵌套映射）。 以下是我们将会使用的示例映射：

```
PUT /my_index
{
    "mappings": {
        "type1" : {
            "properties" : {
                "obj1" : {
                    "type" : "nested"
                }
            }
        }
    }
}
```

这里是一个示例嵌套查询用法：

```
GET /_search
{
    "query": {
        "nested" : {
            "path" : "obj1",
            "score_mode" : "avg",
            "query" : {
                "bool" : {
                    "must" : [
                    { "match" : {"obj1.name" : "blue"} },
                    { "range" : {"obj1.count" : {"gt" : 5}} }
                    ]
                }
            }
        }
    }
}
```

查询path指向嵌套对象路径，该查询包括将在匹配直接路径的嵌套文档上运行的查询，并且加入根父文档。 请注意，查询中引用的任何字段必须使用完整路径（完全限定）。

score_mode允许设置内部children匹配如何影响parent的评分。 它默认为avg，但可以是sum，min，max和none。

还有一个ignore_unmapped选项，当设置为true时，将忽略未映射的path，并且将不匹配此查询的任何文档。 当对可能具有不同映射的多个索引进行查询时，这可能很有用。 当设置为false（默认值）时，如果path未映射，则查询将抛出异常。

自动支持多级嵌套，并进行检测，如果一个嵌套查询存在于另一个嵌套查询中，将产生一个内部嵌套查询自动匹配相关的嵌套级别（而不是root）。

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/5.4/query-dsl-nested-query.html
