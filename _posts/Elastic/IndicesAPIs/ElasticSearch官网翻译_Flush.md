---
title: "ElasticSearch官网翻译_Flush"
date: 2017-06-05 21:39:32
tags: [Elasticsearch]
categories: [Search]
---

## Flush

Flush API允许通过API flush（刷新）一个或多个索引。 索引的flush（刷新）过程基本上通过将数据刷新到索引存储并清除内部transaction log（事务日志）来释放内存。 默认情况下，Elasticsearch使用内存启发式方法，以便根据需要自动触发刷新操作，以清除内存。

```
POST twitter/_flush
```

### Request Parameters（请求参数）

flush API接受以下请求参数：

值|描述
---|---
wait_if_ongoing|如果设置为true，则flush（刷新）操作将阻塞，直到另一个flush（刷新）操作已经执行，才能执行flush。 默认值为false，如果另一个刷新操作已在运行，将导致在分片级别抛出异常。
force|即使不一定需要flush（刷新）也应该强制执行，即如果没有变化将被提交到索引。 即使不存在未提交的更改，事务日志ID也应该递增，这是有用的。 （此设置可视为内部设置）

### Multi Index（多索引）

flush API可以通过单个调用应用于多个索引，甚至可以应用于_all索引。

```
POST kimchy,elasticsearch/_flush

POST _flush
```

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/5.4/indices-flush.html
