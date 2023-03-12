---
title: "ElasticSearch官网翻译_SpanNotQuery"
date: 2017-07-12 17:09:32
tags: [Elasticsearch]
categories: [Search]
---

## Span Not Query

删除与另一个span查询重叠的匹配项。 span not query映射到Lucene的SpanNotQuery。 这是一个例子：

```
GET /_search
{
    "query": {
        "span_not" : {
            "include" : {
                "span_term" : { "field1" : "hoya" }
            },
            "exclude" : {
                "span_near" : {
                    "clauses" : [
                        { "span_term" : { "field1" : "la" } },
                        { "span_term" : { "field1" : "hoya" } }
                    ],
                    "slop" : 0,
                    "in_order" : true
                }
            }
        }
    }
}
```

include和exclude子句可以是任何span类型查询。 include子句是对其匹配进行过滤的跨度查询，并且exclude子句是其匹配不能与返回的匹配重叠的跨度查询。

在上述示例中，除了在他们之前具有la的所有文档，所有具有词条hoya的文档都被过滤。

其他顶级选项：

值|描述
---|---
pre|如果设置在包含范围之前的标记量不能与排除范围重叠。
post|如果设置在包含范围之后的标记量不能与排除范围重叠。
dist|如果设置在包含范围之内的标记量不能与排除范围重叠。 相当于pre和post的设置。

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/5.4/query-dsl-span-not-query.html
