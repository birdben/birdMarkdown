---
title: "ElasticSearch官网翻译_Query"
date: 2017-05-24 14:20:27
tags: [Elasticsearch]
categories: [Search]
---

#### Query

搜索请求正文中的查询元素允许使用Query DSL定义查询。

```
{
    "query" : {
        "term" : { "user" : "kimchy" }
    }
}
```

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/2.4/search-request-query.html

