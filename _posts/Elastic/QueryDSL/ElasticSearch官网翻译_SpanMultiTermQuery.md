---
title: "ElasticSearch官网翻译_SpanMultiTermQuery"
date: 2017-07-12 16:03:51
tags: [Elasticsearch]
categories: [Search]
---

## Span Multi Term Query

span_multi查询允许你包装一个multi term query（wildcard，fuzzy，prefix，range或regexp查询中的一种）为span query，因此可以嵌套。 例：

```
GET /_search
{
    "query": {
        "span_multi":{
            "match":{
                "prefix" : { "user" :  { "value" : "ki" } }
            }
        }
    }
}
```

boost也可以与查询相关联：

```
GET /_search
{
    "query": {
        "span_multi":{
            "match":{
                "prefix" : { "user" :  { "value" : "ki", "boost" : 1.08 } }
            }
        }
    }
}
```

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/5.4/query-dsl-span-multi-term-query.html
