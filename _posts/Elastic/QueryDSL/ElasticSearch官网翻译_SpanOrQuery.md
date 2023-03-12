---
title: "ElasticSearch官网翻译_SpanOrQuery"
date: 2017-07-12 17:03:48
tags: [Elasticsearch]
categories: [Search]
---

## Span Or Query

匹配其跨度子句的联合。 span or query映射到Lucene的SpanOrQuery。 这是一个例子：

```
GET /_search
{
    "query": {
        "span_or" : {
            "clauses" : [
                { "span_term" : { "field" : "value1" } },
                { "span_term" : { "field" : "value2" } },
                { "span_term" : { "field" : "value3" } }
            ]
        }
    }
}
```

clauses元素是一个或多个其他span类型查询的列表。

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/5.4/query-dsl-span-or-query.html
