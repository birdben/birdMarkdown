---
title: "ElasticSearch官网翻译_Explain"
date: 2017-05-26 01:30:27
tags: [Elasticsearch]
categories: [Search]
---

#### Explain

为每次命中结果的得分计算的解释。

```
{
    "explain": true,
    "query" : {
        "term" : { "user" : "kimchy" }
    }
}
```

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/2.4/search-request-explain.html

