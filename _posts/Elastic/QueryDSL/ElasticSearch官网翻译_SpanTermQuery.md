---
title: "ElasticSearch官网翻译_SpanTermQuery"
date: 2017-07-12 16:03:51
tags: [Elasticsearch]
categories: [Search]
---

## Span Term Query

匹配包含词条的跨度。 Span Term Query映射到Lucene的SpanTermQuery。 这是一个例子：

```
GET /_search
{
    "query": {
        "span_term" : { "user" : "kimchy" }
    }
}
```

boost也可以与查询相关联：

```
GET /_search
{
    "query": {
       "span_term" : { "user" : { "value" : "kimchy", "boost" : 2.0 } }
    }
}
```

或者：

```
GET /_search
{
    "query": {
        "span_term" : { "user" : { "term" : "kimchy", "boost" : 2.0 } }
    }
}
```

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/5.4/query-dsl-span-term-query.html
