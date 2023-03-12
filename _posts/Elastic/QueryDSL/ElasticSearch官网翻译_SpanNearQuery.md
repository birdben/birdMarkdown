---
title: "ElasticSearch官网翻译_SpanNearQuery"
date: 2017-07-12 16:47:36
tags: [Elasticsearch]
categories: [Search]
---

## Span Near Query

匹配跨度彼此接近。 可以指定slop，最大间隔不匹配位置数，以及匹配是否需要排序。 span near query映射到Lucene的SpanNearQuery。 这是一个例子：

```
GET /_search
{
    "query": {
        "span_near" : {
            "clauses" : [
                { "span_term" : { "field" : "value1" } },
                { "span_term" : { "field" : "value2" } },
                { "span_term" : { "field" : "value3" } }
            ],
            "slop" : 12,
            "in_order" : false
        }
    }
}
```

clauses元素是一个或多个其他span类型查询的列表，并且slop控制允许的最大间隔不匹配位置数。

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/5.4/query-dsl-span-near-query.html
