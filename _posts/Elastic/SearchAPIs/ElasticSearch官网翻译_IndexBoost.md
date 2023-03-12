---
title: "ElasticSearch官网翻译_IndexBoost"
date: 2017-05-26 01:33:42
tags: [Elasticsearch]
categories: [Search]
---

#### Index Boost

允许在搜索多个索引时为每个索引配置不同升级级别。 这是非常方便的，当来自一个索引的命中比来自另一个索引的命中多（设想社交图，每个用户有一个索引）。

```
{
    "indices_boost" : {
        "index1" : 1.4,
        "index2" : 1.3
    }
}
```

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/2.4/search-request-index-boost.html

