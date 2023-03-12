---
title: "ElasticSearch官网翻译_GetIndex"
date: 2017-06-03 15:09:33
tags: [Elasticsearch]
categories: [Search]
---

## Get Index

get index API允许取回有关一个或多个索引的信息。

```
GET /twitter
```

上面的例子获得了一个称为twitter的索引的信息。 指定索引，别名或通配符表达式是必需的。

get index API也可以通过使用_all或*应用于多个索引或所有索引。

### Filtering index information（过滤索引信息）

get API返回的信息可以过滤，仅包含特定功能，方法是在URL中指定逗号分隔的功能列表：

```
GET twitter/_settings,_mappings
```

上述命令将只返回名为twitter的索引的settings设置和mappings映射。

可用的功能是_settings，_mappings和_aliases。

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/5.4/indices-get-index.html
