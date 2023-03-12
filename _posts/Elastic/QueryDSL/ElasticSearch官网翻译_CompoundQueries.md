---
title: "ElasticSearch官网翻译_CompoundQueries"
date: 2017-06-29 17:19:18
tags: [Elasticsearch]
categories: [Search]
---

## Compound queries（复合查询）

复合查询包装其他复合或叶子查询，以组合其结果和分数，以更改其行为，或从查询切换到过滤器上下文。

该组中的查询是：

- constant_score query

一个包装另一个查询但是在过滤器上下文中执行的查询。 所有匹配的文档都被赋予相同的“常量“_score。

- bool query

默认查询组合多个叶子或复合查询子句，如must，should，must_not或filter子句。 must和should子句的分数结合在一起，越多的匹配子句越好，而must_not和filter子句在过滤器上下文中执行。

- dis_max query

一个接受多个查询的查询，并返回与任何查询子句匹配的文档。 当bool查询组合来自所有match查询的分数时，dis_max查询使用单个最佳匹配查询子句的得分。

- function_score query

使用函数修改主查询返回的分数，以考虑诸如流行度，新近度，距离或通过脚本实现的自定义算法等因素。

- boosting query

返回与positive查询相符的文档，但是降低了与negative查询匹配的文档的分数。

- indices query

对指定的索引执行一个查询，另一个用于其他索引。

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/5.4/compound-queries.html
