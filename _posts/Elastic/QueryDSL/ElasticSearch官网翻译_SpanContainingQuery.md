---
title: "ElasticSearch官网翻译_SpanContainingQuery"
date: 2017-07-12 17:50:52
tags: [Elasticsearch]
categories: [Search]
---

## Span Containing Query

返回包含另一个span查询的匹配。 span containing query映射到Lucene的SpanContainingQuery。 这是一个例子：

```
GET /_search
{
    "query": {
        "span_containing" : {
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

big和little的子句可以是任何span类型查询。 从big的匹配跨度中包含匹配little的被返回。

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/5.4/query-dsl-span-containing-query.html
