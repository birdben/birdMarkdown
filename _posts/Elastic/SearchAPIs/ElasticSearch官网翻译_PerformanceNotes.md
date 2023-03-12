---
title: "ElasticSearch官网翻译_PerformanceNotes"
date: 2017-05-28 14:23:09
tags: [Elasticsearch]
categories: [Search]
---

#### Performance Notes

与任何剖析器一样，Profile API引入了一个不可忽略的查询执行开销。 测量底层方法调用（例如advance和next_doc）的行为可能相当昂贵，因为这些方法在紧密循环中调用。 因此，默认情况下，不应在生产设置中启用profiling概要分析，并且不应与non-profiled非剖析查询时间进行比较。 Profiling剖析只是一个诊断工具。

还有一些特殊的Lucene优化被禁用，因为它们不适合进行profiling剖析。 这可能会导致一些查询报告更大的相对时间比不使用剖析，但与剖析查询中的其他组件相比，通常不会产生剧烈的影响。

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/2.4/_performance_notes.html

