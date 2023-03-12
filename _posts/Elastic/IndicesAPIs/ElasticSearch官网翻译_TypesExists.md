---
title: "ElasticSearch官网翻译_TypesExists"
date: 2017-06-03 17:44:17
tags: [Elasticsearch]
categories: [Search]
---

## Types Exists

用于检查index/indices中是否存在type/types。

```
HEAD twitter/_mapping/tweet
```

HTTP状态代码指示类型是否存在。 404意味着它不存在，而200表示它不存在。

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/5.4/indices-types-exists.html
