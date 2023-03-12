---
title: "ElasticSearch官网翻译_CollectorsSection"
date: 2017-05-28 13:50:55
tags: [Elasticsearch]
categories: [Search]
---

#### collectors Section

响应的收集器部分显示上层执行细节。 Lucene通过定义一个"Collector"负责匹配文档的遍历，计分和收集来工作的。 收集者也是单个查询如何记录聚合结果，执行无范围的"global"查询，执行post-query filter后期查询过滤器等。

看前面的例子：

```
"collector": [
    {
       "name": "SimpleTopScoreDocCollector",
       "reason": "search_top_hits",
       "time": "2.206529000ms"
    }
]
```

我们看到一个名为SimpleTopScoreDocCollector的收集器。 这是Elasticsearch使用的默认"评分和排序"收集器。 "reason"字段尝试给出类名的简单英文描述。 "time"类似于Query tree中的时间：包含所有children的时间。 同样，children可以列出所有子收集器。

应该注意的是，收集器时间独立于查询时间。 它们被独立计算，组合和归一化！ 由于Lucene执行的性质，将Collectors收集者的时间"merge"到Query section查询部分是不可能的，因此它们分开显示。

作为参考，各种收集器的理由是：

值|描述
---|---
search_sorted|一个收集器对文档进行分类和排序。 这是最常见的收集器，将在大多数简单的搜索中看到
search_count|只能计算与查询匹配的文档数量但不能获取源的收集器。 当指定size:0或search_type=count时，会出现这种情况
search_terminate_after_count|在找到n个匹配文档之后终止搜索执行的收集器。 当指定了terminate_after_count查询参数时，可以看到这一点
search_min_score|只能返回具有分数大于n的匹配文档的收集器。 当指定顶级参数min_score时，会看到这一点。
search_multi|收集器包装了其他几个收集器。 当搜索，聚合，全局agg和post_filters的组合在单个搜索中组合时，就会看到这一点。
search_timeout|一个收集器在一段指定的时间后停止执行。 当指定timeout顶级参数时，会看到这一点。
aggregation|Elasticsearch用于针对查询范围运行聚合的收集器。 单个aggregation收集器用于收集所有聚合的文档，因此你将在名称中看到聚合列表。
global_aggregation|针对全局查询范围而不是指定查询执行聚合的收集器。 因为全局范围与执行的查询必然不同，所以它必须执行它自己的match_all查询（你将看到添加到Query section查询部分）来收集整个数据集

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/2.4/_literal_collectors_literal_section.html

