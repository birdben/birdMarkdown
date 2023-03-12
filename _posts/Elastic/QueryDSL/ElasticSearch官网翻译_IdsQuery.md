---
title: "ElasticSearch官网翻译_IdsQuery"
date: 2017-06-29 15:54:51
tags: [Elasticsearch]
categories: [Search]
---

## Ids Query

过滤只提供ids的文档。 请注意，此查询使用_uid字段。

```
GET /_search
{
    "query": {
        "ids" : {
            "type" : "my_type",
            "values" : ["1", "4", "100"]
        }
    }
}
```

type是可选的，可以省略，也可以接受一个值数组。 如果没有指定type，则会尝试在索引映射中定义的所有类型。

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/5.4/query-dsl-ids-query.html
