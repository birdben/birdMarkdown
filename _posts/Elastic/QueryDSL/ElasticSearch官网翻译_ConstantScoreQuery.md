---
title: "ElasticSearch官网翻译_ConstantScoreQuery"
date: 2017-06-29 18:03:51
tags: [Elasticsearch]
categories: [Search]
---

## Constant Score Query

一个包装另一个查询的查询，简单的返回一个常量分数等于过滤器中每个文档的查询提升。 对应到Lucene ConstantScoreQuery。

```
GET /_search
{
    "query": {
        "constant_score" : {
            "filter" : {
                "term" : { "user" : "kimchy"}
            },
            "boost" : 1.2
        }
    }
}
```

过滤器子句在filter context（过滤器上下文）中执行，这意味着计分被忽略，并且子句被考虑用于缓存。

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/5.4/query-dsl-constant-score-query.html
