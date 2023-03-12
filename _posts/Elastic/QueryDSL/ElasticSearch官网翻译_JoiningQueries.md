---
title: "ElasticSearch官网翻译_JoiningQueries"
date: 2017-06-30 15:51:29
tags: [Elasticsearch]
categories: [Search]
---

## Joining queries（连接查询）

在像Elasticsearch这样的分布式系统中执行full SQL-style连接操作是非常昂贵的。 相反，Elasticsearch提供了两种形式的连接，其设计可以水平扩展。

- nested query（嵌套查询）

文档可能包含nested类型的字段。 这些字段用于索引对象数组，其中每个对象可以被查询（使用nested query嵌套查询）作为独立文档。

- has_child和has_parent queries

parent-child relationship（父子关系）可以存在于单个索引中的两个文档类型之间。 has_child查询返回父文档，其子文档与指定的查询匹配，而has_parent查询返回子文档，其父文档与指定的查询匹配。

另请参阅terms query中的terms-lookup mechanism，它允许你从另一个文档中包含的值构建terms query。

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/5.4/joining-queries.html
