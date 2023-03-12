---
title: "ElasticSearch官网翻译_DeleteIndex"
date: 2017-06-03 15:05:51
tags: [Elasticsearch]
categories: [Search]
---

## Delete Index

delete index API可以删除现有的索引。

```
DELETE /twitter
```

上面的例子删除了一个叫做twitter的索引。 指定索引，别名或通配符表达式是必需的。

delete index API也可以通过使用逗号分隔列表应用于多个索引或者使用_all或*应用于所有索引（请注意！）。

为了禁用通过通配符或者_all删除索引，请将config中的action.destructive_requires_name设置设置为true。 也可以通过集群update setttings api来更改此设置。

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/5.4/indices-delete-index.html
