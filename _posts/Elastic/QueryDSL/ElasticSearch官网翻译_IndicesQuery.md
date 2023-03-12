---
title: "ElasticSearch官网翻译_IndicesQuery"
date: 2017-06-30 15:40:18
tags: [Elasticsearch]
categories: [Search]
---

## Indices Query

###### 警告：5.0.0中弃用

> 使用在_index字段上搜索替代

在跨多个索引执行搜索的情况下，indices query很有用。 它允许指定索引名称列表和内部查询，该内部查询仅针对与该列表上的名称匹配的索引执行。 对于搜索的其他索引，但是与列表中的（索引）条目不匹配，则会执行替代的no_match_query。

```
GET /_search
{
    "query": {
        "indices" : {
            "indices" : ["index1", "index2"],
            "query" : { "term" : { "tag" : "wow" } },
            "no_match_query" : { "term" : { "tag" : "kow" } }
        }
    }
}
```

你可以使用index字段提供单个索引。

no_match_query也可以设置为"string"字符串值为none（不匹配任何文档）和all（匹配所有文档）。 默认为all。

query是必需的，以及indices（或index）。

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/5.4/query-dsl-indices-query.html
