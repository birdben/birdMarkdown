---
title: "ElasticSearch官网翻译_TermLevelQueries"
date: 2017-06-22 10:24:35
tags: [Elasticsearch]
categories: [Search]
---

## Term level queries

虽然full text queries（全文查询）将在执行之前分析查询字符串，但term-level queries（词条级别查询）将根据存储在倒排索引中的确切词条来进行操作。

这些查询通常用于结构化数据，如数字，日期和枚举，而不是全文本字段。 或者，它们允许你使用低级查询，如前面描述的分析过程。

该组中的查询是：

- term query

在指定字段中查找包含指定的确切词条的文档。

- terms query

在指定字段中查找包含指定的任何确切词条的文档。

- range query

在指定字段中查找包含指定范围内的值（日期，数字或字符串）的文档。

- exists query

在指定的字段中查找包含任何non-null value（非空值）的文档。

- prefix query

在指定字段中查找包含以指定的确切前缀开头的词条的文档。

- wildcard query

在指定字段中查找包含与指定的模式匹配的词条的文档，其中模式支持单字符通配符(?)和多字符通配符(*)

- regexp query

在指定字段中查找包含与指定的正则表达式匹配的词条的文档。

- fuzzy query

在指定字段中查找包含与指定词条模糊相似的词条的文档。 模糊性计量单位为1或2的Levenshtein edit distance。

- type query

查找指定类型的文档。

- ids query

查找具有指定类型和ID的文档。

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/5.4/term-level-queries.html
