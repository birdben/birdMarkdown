---
title: "ElasticSearch官网翻译_GetMapping"
date: 2017-06-03 17:21:55
tags: [Elasticsearch]
categories: [Search]
---

## Get Mapping

Get mapping API允许获取index或index/type的映射定义。

```
GET /twitter/_mapping/tweet
```

### Multiple Indices and Types（多索引和类型）

Get mapping API可用于通过单个调用获取多个索引或类型映射。 API的一般用法遵循以下语法：host:port/{index}/_mapping/{type}其中{index}和{type}都可以接受以逗号分隔的名称列表。 要获取所有索引的映射，你可以使用_all替换{index}。 以下是一些例子：

```
GET /_mapping/tweet,kimchy

GET /_all/_mapping/tweet,book
```

如果要获取所有索引和类型的映射，则以下两个示例是等价的：

```
GET /_all/_mapping

GET /_mapping
```

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/5.4/indices-get-mapping.html
