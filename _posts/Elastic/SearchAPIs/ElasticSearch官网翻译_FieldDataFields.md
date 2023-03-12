---
title: "ElasticSearch官网翻译_FieldDataFields"
date: 2017-05-25 00:40:00
tags: [Elasticsearch]
categories: [Search]
---

#### Field Data Fields

允许返回每个命中的字段的字段数据表示，例如：

```
{
    "query" : {
        ...
    },
    "fielddata_fields" : ["test1", "test2"]
}
```

字段的数据字段可以在不存储的字段上工作。

重要的是要明白，使用fielddata_fields参数将导致该字段的terms词条被加载到内存（缓存），这将导致更多的内存消耗。

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/2.4/search-request-fielddata-fields.html
