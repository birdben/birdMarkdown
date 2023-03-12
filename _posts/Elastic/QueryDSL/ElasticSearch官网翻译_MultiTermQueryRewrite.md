---
title: "ElasticSearch官网翻译_MultiTermQueryRewrite"
date: 2017-07-12 20:05:31
tags: [Elasticsearch]
categories: [Search]
---

## Multi Term Query Rewrite

Multi term query多项查询（如wildcard通配符和prefix前缀）称为多项查询，最终将经历一个重写过程。这也发生在query_string上。所有这些查询都允许使用rewrite参数控制如何重写它们：

- constant_score（默认）：一种重写方法，当几乎没有匹配的词条时执行像constant_score_boolean，否则按顺序访问所有匹配的词条并标记该词条的文档。匹配文档被赋予等于查询boost的常量分数。

- scoring_boolean：一种重写方法，首先将每个词条翻译成一个布尔查询中的一个should子句，并保持按查询计算的分数。请注意，通常，这些分数对用户无意义，并且需要不平凡的CPU进行计算，因此使用constant_score_auto几乎总是更好。如果超出布尔查询限制（默认为1024），则此重写方法将会触发太多的子句失败。

- constant_score_boolean：与scoring_boolean类似，除非不计算分数。相反，每个匹配的文档会收到等于查询boost的常量分数。如果超出布尔查询限制（默认为1024），则此重写方法将会触发太多的子句失败。

- top_terms_N：一种重写方法，首先将每个词条翻译成布尔查询中的should子句，并保持由查询计算的分数。这个重写方法只使用最高的评分项，所以它不会超出布尔查询最大子句数量。 N控制要使用的最高评分项的大小。

- top_terms_boost_N：一种重写方法，首先将每个词条翻译成布尔查询中的should子句，但分数仅计算为boost。此重写方法仅使用最高的评分项，所以它不会超出布尔查询最大子句数量。 N控制要使用的最高评分项的大小。

- top_terms_blended_freqs_N：一种重写方法，首先将每个词条转换为布尔查询中的should子句，但所有term查询都会计算分数，就好像它们具有相同的频率。实际上，使用的频率是所有匹配词条的最大频率。这个重写方法只使用最高的评分项，所以它不会超出布尔查询最大子句数量。 N控制要使用的最高评分项的大小。

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/5.4/query-dsl-multi-term-rewrite.html
