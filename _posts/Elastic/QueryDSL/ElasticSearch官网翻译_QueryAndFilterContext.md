---
title: "ElasticSearch官网翻译_QueryAndFilterContext"
date: 2017-06-07 18:48:17
tags: [Elasticsearch]
categories: [Search]
---

## Query and filter context

查询子句的行为取决于它是在query context（查询上下文）还是在filter context（过滤器上下文）中使用：

- Query context（查询上下文）

查询语句中使用的查询上下文回答了“这个文档与这个查询子句的匹配程度如何？”的问题。除了决定文档是否匹配之外，查询子句还会计算一个_score，相对于其他文档，文档的匹配程度如何。

每当将查询子句传递query参数（如search API中的query参数）时，查询上下文才会生效。

- Filter context（过滤上下文）

在过滤器上下文中，查询子句回答了“这个文档是否符合这个查询子句？”问题，回答是简单的是或否 - 不计算分数。过滤器上下文主要用于过滤结构化数据，例如

 - 这个timestamp是否在2015年至2016年之间？
 - status字段是否设置为"published"？

经常使用的过滤器将被Elasticsearch自动缓存，以加快性能。

当查询子句传递filter参数（如bool查询中的filter或must_not参数，constant_score查询中的filter参数或者filter聚合）时，过滤器上下文将生效。

以下是在search API中在查询上下文和过滤器上下文中使用的查询子句的示例。此查询将匹配满足以下所有条件的文档：

- title字段包含单词search。
- content字段包含单词elasticsearch。
- status字段包含确切的单词published。
- publish_date字段包含从2015年1月1日起的日期。

```
GET /_search
{
  "query": { (1)
    "bool": { (2)
      "must": [
        { "match": { "title":   "Search"        }}, (3)
        { "match": { "content": "Elasticsearch" }}  (4)
      ],
      "filter": [ (5)
        { "term":  { "status": "published" }}, (6)
        { "range": { "publish_date": { "gte": "2015-01-01" }}} (7)
      ]
    }
  }
}
```

- (1) query参数表示查询上下文。
- (2)(3)(4) bool和两个match子句用于查询上下文中，这意味着它们用于评估每个文档匹配的程度。
- (5) filter参数表示过滤器上下文。
- (6)(7) term和range子句用于过滤器上下文中。 他们会过滤掉不符合的文档，但不会影响匹配文档的分数。

###### 提示

> 在查询上下文中使用查询子句，这些条件会影响匹配文档的分数（即文档的匹配程度），并且使用过滤器上下文中的所有其他查询子句。

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/5.4/query-filter-context.html
