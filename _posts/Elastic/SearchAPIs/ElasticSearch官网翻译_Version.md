---
title: "ElasticSearch官网翻译_Version"
date: 2017-05-26 01:32:11
tags: [Elasticsearch]
categories: [Search]
---

#### Version

返回每个搜索命中的版本。

```
{
    "version": true,
    "query" : {
        "term" : { "user" : "kimchy" }
    }
}
```

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/2.4/search-request-version.html

