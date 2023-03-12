---
title: "ElasticSearch官网翻译_QueryDSL"
date: 2017-06-07 18:37:55
tags: [Elasticsearch]
categories: [Search]
---

## Query DSL

Elasticsearch提供基于JSON的完整Query DSL来定义查询。 将Query DSL视为查询的AST，由两种类型的子句组成：

- Leaf query clauses（叶子查询子句）

Leaf query clauses（叶子查询子句）在特定字段中查找特定值，例如match，term或range查询。 这些查询可以自己使用。

- Compound query clauses（复合查询子句）

Compound query clauses（复合查询子句）包装其他叶子或复合查询，并用于以逻辑方式（如bool或dis_max查询）组合多个查询，或者更改其行为（例如constant_score查询）。

查询子句的行为有所不同，具体取决于它们是在query context or filter context（查询上下文还是过滤器上下文）中使用。

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/5.4/query-dsl.html
