---
title: "ElasticSearch官网翻译_SpanQueries"
date: 2017-07-12 14:43:29
tags: [Elasticsearch]
categories: [Search]
---

## Span queries（跨度查询）

跨度查询是低级位置查询，可以提供对指定词条的顺序和接近度的专家控制。这些通常用于对法律文件或专利执行非常专业的查询。

span query跨度查询不能与non-span query非跨查询混合（除了span_multi查询外）。

该组中的查询是：

- span_term query

等同于term query，但与其他跨查询一起使用。

- span_multi query

包含term，range，prefix，wildcard，regexp或fuzzy查询。

- span_first query

接受另一个跨度查询，其匹配必须出现在字段的前N个位置。

- span_near query

接受多个跨度查询，其匹配必须在彼此的指定距离内，并且可能以相同的顺序。

- span_or query

组合多个跨度查询 - 返回与任何指定查询匹配的文档。

- span_not query

包装另一个跨度查询，并排除与该查询匹配的任何文档。

- span_containing query

接受跨度查询的列表，但只返回也匹配第二个跨度查询的跨度。

- span_within query

单个跨度查询的结果就会被返回，只要其跨度位于由其他跨度查询的列表返回的跨度内。

- field_masking_span query

允许跨越不同字段的span-near或span-or的查询。

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/5.4/span-queries.html
