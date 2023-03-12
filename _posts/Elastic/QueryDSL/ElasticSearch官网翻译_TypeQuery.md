---
title: "ElasticSearch官网翻译_TypeQuery"
date: 2017-06-29 15:54:51
tags: [Elasticsearch]
categories: [Search]
---

## Type Query

过滤与提供的文档/映射类型匹配的文档。

```
GET /_search
{
    "query": {
        "type" : {
            "value" : "my_type"
        }
    }
}
```

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/5.4/query-dsl-type-query.html
