---
title: "ElasticSearch官网翻译_SpanFirstQuery"
date: 2017-07-12 16:27:57
tags: [Elasticsearch]
categories: [Search]
---

## Span First Query

匹配跨度一个字段的开头附近。 span first query映射到Lucene的SpanFirstQuery。 这是一个例子：

```
GET /_search
{
    "query": {
        "span_first" : {
            "match" : {
                "span_term" : { "user" : "kimchy" }
            },
            "end" : 3
        }
    }
}
```

match子句可以是任何其他span类型查询。 end控制匹配中允许的最大终点位置。

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/5.4/query-dsl-span-first-query.html
