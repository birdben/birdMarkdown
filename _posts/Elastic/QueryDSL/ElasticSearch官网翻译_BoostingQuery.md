---
title: "ElasticSearch官网翻译_BoostingQuery"
date: 2017-06-30 15:34:25
tags: [Elasticsearch]
categories: [Search]
---

## Boosting Query

boosting查询可用于有效降级给定查询匹配的结果。 与bool查询中的"NOT"子句不同，它仍然选择包含不需要的词条的文档，但是减少了它们的总体得分。

```
GET /_search
{
    "query": {
        "boosting" : {
            "positive" : {
                "term" : {
                    "field1" : "value1"
                }
            },
            "negative" : {
                 "term" : {
                     "field2" : "value2"
                }
            },
            "negative_boost" : 0.2
        }
    }
}
```

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/5.4/query-dsl-boosting-query.html
