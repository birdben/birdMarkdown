---
title: "ElasticSearch官网翻译_WildcardQuery"
date: 2017-06-28 17:57:40
tags: [Elasticsearch]
categories: [Search]
---

## Wildcard Query

匹配文档的字段匹配wildcard通配符表达式（not analyzed 未分析）。 支持的通配符是"*"，它匹配任何字符序列（包括空字符序列）和"?"，它匹配任何单个字符。 请注意，这个查询可能很慢，因为它需要迭代许多词条。 为了防止非常慢的通配符查询，通配符词条不能以"*"或"?"中的任何一个通配符开头。 通配符查询对应Lucene的WildcardQuery。

```
GET /_search
{
    "query": {
        "wildcard" : { "user" : "ki*y" }
    }
}
```

boost也可以与查询相关联：

```
GET /_search
{
    "query": {
        "wildcard" : { "user" : { "value" : "ki*y", "boost" : 2.0 } }
    }
}
```

或者：

```
GET /_search
{
    "query": {
        "wildcard" : { "user" : { "wildcard" : "ki*y", "boost" : 2.0 } }
    }
}
```

这个multi term query允许你使用rewrite参数来控制如何重写它。

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/5.4/query-dsl-wildcard-query.html
