---
title: "ElasticSearch官网翻译_HasParentQuery"
date: 2017-06-30 17:28:11
tags: [Elasticsearch]
categories: [Search]
---

## Has Parent Query

has_parent查询接受一个查询和parent type（父类型）。 该查询在父文档空间中执行，该父文档空间由parent type（父类型）指定。 此查询返回与相关父文档匹配的子文档。 对于其余的has_parent查询具有与has_child查询相同的选项和相同的方式工作。

```
GET /_search
{
    "query": {
        "has_parent" : {
            "parent_type" : "blog",
            "query" : {
                "term" : {
                    "tag" : "something"
                }
            }
        }
    }
}
```

#### Scoring capabilities（评分功能）

has_parent也支持评分。 默认值为false，忽略父文档的分数。 在这种情况下，分数等于对has_parent查询的boost提升（默认为1）。 如果分数设置为true，则将匹配的父文档的分数聚合到属于匹配父文档的子文档中。 可以使用has_parent查询中的score字段指定评分模式：

```
GET /_search
{
    "query": {
        "has_parent" : {
            "parent_type" : "blog",
            "score" : true,
            "query" : {
                "term" : {
                    "tag" : "something"
                }
            }
        }
    }
}
```

#### Ignore Unmapped（忽略未映射的）

当设置为true时，ignore_unmapped选项将忽略未映射的type，并且将不会匹配此查询的任何文档。 当对可能具有不同映射的多个索引进行查询时，这可能很有用。 当设置为false（默认值）时，如果type未映射，则查询将抛出异常。

#### Sorting（排序）

子文档不能通过常规排序选项在匹配父文档中的字段中进行排序。 如果你需要在父文档中按字段排序子文档，那么你可以使用function_score查询，然后按_score进行排序。

通过父文档的view_count字段排序tags

```
GET /_search
{
    "query": {
        "has_parent" : {
            "parent_type" : "blog",
            "score" : true,
            "query" : {
                "function_score" : {
                    "script_score": {
                        "script": "_score * doc['view_count'].value"
                    }
                }
            }
        }
    }
}
```

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/5.4/query-dsl-has-parent-query.html
