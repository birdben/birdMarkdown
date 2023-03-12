---
title: "ElasticSearch官网翻译_HasChildQuery"
date: 2017-06-30 16:56:54
tags: [Elasticsearch]
categories: [Search]
---

## Has Child Query

has_child过滤器接受一个查询和child type（子类型）来执行，返回结果的父文档具有与查询匹配的子文档。 这是一个例子：

```
GET /_search
{
    "query": {
        "has_child" : {
            "type" : "blog_tag",
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

has_child还支持评分。 支持的评分模式是min，max，sum，avg或none。 默认值为none，并产生与以前版本相同的行为。 如果评分模式被设置为不是none，则所有匹配的子文档的分数被聚合到相关联的父文档中。 可以使用has_child查询中的score_mode字段指定评分类型：

```
GET /_search
{
    "query": {
        "has_child" : {
            "type" : "blog_tag",
            "score_mode" : "min",
            "query" : {
                "term" : {
                    "tag" : "something"
                }
            }
        }
    }
}
```

#### Min/Max Children

has_child查询允许你指定父文档要求匹配子文档数量的最小值和（或）最大值，才被认为是匹配的：

```
GET /_search
{
    "query": {
        "has_child" : {
            "type" : "blog_tag",
            "score_mode" : "min",
            "min_children": 2, (1)
            "max_children": 10, (2)
            "query" : {
                "term" : {
                    "tag" : "something"
                }
            }
        }
    }
}
```

- (1)(2) min_children和max_children都是可选的。

min_children和max_children参数可以与score_mode参数组合。

#### Ignore Unmapped（忽略未映射的）

当设置为true时，ignore_unmapped选项将忽略未映射的type，并且将不匹配此查询的任何文档。 当对可能具有不同映射的多个索引进行查询时，这可能很有用。 当设置为false（默认值）时，如果type未映射，则查询将抛出异常。

#### Sorting（排序）

父文档不能通过常规排序选项在匹配的子文档中的字段进行排序。 如果你需要在子文档中按字段排序父文档，则可以使用function_score查询，然后按_score进行排序。

按子文档的click_count字段排序blogs：

```
GET /_search
{
    "query": {
        "has_child" : {
            "type" : "blog_tag",
            "score_mode" : "max",
            "query" : {
                "function_score" : {
                    "script_score": {
                        "script": "_score * doc['click_count'].value"
                    }
                }
            }
        }
    }
}
```

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/5.4/query-dsl-nested-query.html
