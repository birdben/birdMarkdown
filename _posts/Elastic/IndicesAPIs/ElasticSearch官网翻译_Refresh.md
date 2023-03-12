---
title: "ElasticSearch官网翻译_Refresh"
date: 2017-06-06 01:18:32
tags: [Elasticsearch]
categories: [Search]
---

## Refresh

Refresh API允许显式刷新一个或多个索引，使自上次刷新以来执行的所有操作可用于搜索。 （近）实时功能取决于所使用的索引引擎。 例如，一个内部请求需要refresh（刷新）被调用，但默认情况下会定期进行刷新。

```
POST /twitter/_refresh
```

### Multi Index（多索引）

Refresh API可以通过单个调用应用于多个索引，甚至可以应用于_all所有索引。

```
POST /kimchy,elasticsearch/_refresh

POST /_refresh
```

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/5.4/indices-refresh.html
