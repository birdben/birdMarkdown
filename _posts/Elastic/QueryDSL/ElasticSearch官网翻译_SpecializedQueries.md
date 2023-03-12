---
title: "ElasticSearch官网翻译_SpecializedQueries"
date: 2017-07-12 14:33:10
tags: [Elasticsearch]
categories: [Search]
---

## Specialized queries（专业查询）

该组包含不适用其他组的查询：

- more_like_this query（相似查询）

此查询查找与指定的文本，文档或文档集合相似的文档。

- template query（模板查询）

template query接受一个Mustache模板（内联，索引或从一个文件）和一个参数映射，并组合这两者生成要执行的最终查询。

- script query（脚本查询）

此查询允许脚本充当过滤器。 另请参阅function_score query。

- percolate query（渗透查询）

此查询查找存储为与指定文档匹配的文档的查询。

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/5.4/specialized-queries.html
