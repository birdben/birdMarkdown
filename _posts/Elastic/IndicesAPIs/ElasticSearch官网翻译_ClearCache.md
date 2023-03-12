---
title: "ElasticSearch官网翻译_ClearCache"
date: 2017-06-05 20:22:04
tags: [Elasticsearch]
categories: [Search]
---

## Clear Cache

Clear cache API允许清除与一个或多个索引相关联的所有缓存或特定缓存。

```
POST /twitter/_cache/clear
```

默认情况下，API将清除所有缓存。 通过设置query，fielddata或request可以明确地清除特定的缓存。

与特定字段相关的所有缓存也可以通过使用相关字段的逗号分隔列表指定fields参数来清除。

### Multi Index（多索引）

Clear cache API可以通过单个调用应用于多个索引，或甚至应用于_all索引。

```
POST /kimchy,elasticsearch/_cache/clear

POST /_cache/clear
```

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/5.4/indices-clearcache.html
