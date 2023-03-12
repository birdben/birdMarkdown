---
title: "ElasticSearch官网翻译_RewriteSection"
date: 2017-05-28 14:08:31
tags: [Elasticsearch]
categories: [Search]
---

#### rewrite Section

Lucene的所有查询都经历了"rewriting"过程。 查询（及其子查询）可能会被重写一次或多次，并且该过程继续，直到查询停止更改。 此过程允许Lucene执行优化，例如删除冗余子句，替换一个查询以获得更有效的执行路径等。例如，Boolean → Boolean → TermQuery可以重写为TermQuery，因为在这种情况下所有布尔值都是不必要的。

重写过程复杂且难以显示，因为查询可能会发生巨大变化。 而不是显示中间结果，总重写时间只是简单的显示为一个值（以纳秒为单位）。 该值是累积的，并且包含所有重写的查询的总时间。

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/2.4/_literal_rewrite_literal_section.html

