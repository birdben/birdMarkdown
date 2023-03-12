---
title: "ElasticSearch官网翻译_ExistsQuery"
date: 2017-06-28 17:29:08
tags: [Elasticsearch]
categories: [Search]
---

## Exists Query

返回原始字段中至少有一个non-null非空值的文档：

```
GET /_search
{
    "query": {
        "exists" : { "field" : "user" }
    }
}
```

例如，这些文档将全部匹配上述查询：

```
{ "user": "jane" }
{ "user": "" } (1)
{ "user": "-" } (2)
{ "user": ["jane"] }
{ "user": ["jane", null ] } (3)
```

- (1) 空字符串是非空值。
- (2) 即使standard标准分析器将排出0个词元，原始字段为non-null非空。
- (3) 至少需要一个非空值。

这些文档不匹配上述查询：

```
{ "user": null }
{ "user": [] } (1)
{ "user": [null] } (2)
{ "foo":  "bar" } (3)
```

- (1) 该字段没有值。
- (2) 至少需要一个非空值。
- (3) 没有user字段。

#### null_value mapping

如果字段映射包括null_value设置，那么显式的null值将被替换为指定的null_value。 例如，如果user字段被映射如下：

```
  "user": {
    "type": "keyword",
    "null_value": "_null_"
  }
```

那么显式的null值将被索引为字符串_null_，并且以下文档将与exists过滤器匹配：

```
{ "user": null }
{ "user": [null] }
```

但是，这些文档（没有明确的null值）在user字段中仍然没有值，因此与exists过滤器不匹配：

```
{ "user": [] }
{ "foo": "bar" }
```

#### missing query

missing查询已被删除，因为它可以有利地被替换为一个exists查询在一个must_not子句内，如下：

```
GET /_search
{
    "query": {
        "bool": {
            "must_not": {
                "exists": {
                    "field": "user"
                }
            }
        }
    }
}
```

此查询返回user字段中没有值的文档。

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/5.4/query-dsl-exists-query.html
