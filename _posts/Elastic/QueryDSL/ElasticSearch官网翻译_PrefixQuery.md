---
title: "ElasticSearch官网翻译_PrefixQuery"
date: 2017-06-28 17:51:21
tags: [Elasticsearch]
categories: [Search]
---

## Prefix Query

匹配文档的字段包含具有指定前缀（not analyzed 未分析）的词条。 前缀查询映射到Lucene PrefixQuery。 以下匹配user字段包含以ki开头的词条的文档：

```
GET /_search
{ "query": {
    "prefix" : { "user" : "ki" }
  }
}
```

boost也可以与查询相关联：

```
GET /_search
{ "query": {
    "prefix" : { "user" :  { "value" : "ki", "boost" : 2.0 } }
  }
}
```

或使用prefix语法，在5.0.0中弃用：

```
GET /_search
{ "query": {
    "prefix" : { "user" :  { "prefix" : "ki", "boost" : 2.0 } }
  }
}
```

这个multi term query允许你使用rewrite参数来控制如何重写它。

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/5.4/query-dsl-prefix-query.html
