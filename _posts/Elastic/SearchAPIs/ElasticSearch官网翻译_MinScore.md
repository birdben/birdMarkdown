---
title: "ElasticSearch官网翻译_MinScore"
date: 2017-05-26 01:39:53
tags: [Elasticsearch]
categories: [Search]
---

#### min_score

排除_score小于min_score中指定的最小值的文档：

```
{
    "min_score": 0.5,
    "query" : {
        "term" : { "user" : "kimchy" }
    }
}
```

请注意，大多数情况下，这并不太有意义，但适用于高级用例。

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/2.4/search-request-min-score.html

