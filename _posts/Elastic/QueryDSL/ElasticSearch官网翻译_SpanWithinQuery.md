---
title: "ElasticSearch官网翻译_SpanWithinQuery"
date: 2017-07-12 17:59:06
tags: [Elasticsearch]
categories: [Search]
---

## Span Within Query

返回被另一个span查询包含的匹配。 查询内的范围映射到Lucene SpanWithinQuery。 这是一个例子：

```
GET /_search
{
    "query": {
        "span_within" : {
            "little" : {
                "span_term" : { "field1" : "foo" }
            },
            "big" : {
                "span_near" : {
                    "clauses" : [
                        { "span_term" : { "field1" : "bar" } },
                        { "span_term" : { "field1" : "baz" } }
                    ],
                    "slop" : 5,
                    "in_order" : true
                }
            }
        }
    }
}
```

big和little的子句可以是任何span类型查询。 从little的匹配跨度中被包含匹配big的被返回。

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/5.4/query-dsl-span-within-query.html
