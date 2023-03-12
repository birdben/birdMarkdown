---
title: "ElasticSearch官网翻译_ProfileAPI"
date: 2017-05-28 12:45:51
tags: [Elasticsearch]
categories: [Search]
---

#### Profile API

###### 警告

此功能是实验性的，可能会在将来的版本中完全更改或删除。 Elastic将采取最大的努力来解决任何问题，但实验功能不受SLA官方GA功能的支持。

Profile API提供有关查询中各个组件的执行的详细时间信息。 用户可以深入了解如何在底层执行查询，以便用户可以理解为什么某些查询较慢，并采取措施来改进其慢速查询。

Profile API的输出是非常冗长的，特别是对于在许多分片上执行的复杂查询。 推荐格式化打印响应，帮助了解输出

###### 注意

Profile API提供的详细信息直接暴露了Lucene类的名称和概念，这意味着对结果的完整解释需要相当先进的Lucene知识。 这个页面试图给出一个关于Lucene如何执行查询的崩溃例子，以便你可以使用Profile API来成功诊断和调试查询，但这只是一个概述。 为了完整的了解，请参考Lucene的文档以及代码。

就这样说，通常不需要完整的理解来修正缓慢的查询。 例如，通常看到查询的特定组件缓慢是足够的，而且不需要理解为什么该查询的advance阶段是原因。

##### 用法

可以通过添加顶级profile参数来剖析任何_search请求：

```
curl -XGET 'localhost:9200/_search' -d '{
  "profile": true,
  "query" : {
    "match" : { "message" : "search test" }
  }
}
```

- (1) 将顶级profile参数设置为true将启用搜索剖析

这将产生以下结果：

```
{
   "took": 25,
   "timed_out": false,
   "_shards": {
      "total": 1,
      "successful": 1,
      "failed": 0
   },
   "hits": {
      "total": 1,
      "max_score": 1,
      "hits": [ ... ] (1)
   },
   "profile": {
     "shards": [
        {
           "id": "[htuC6YnSSSmKFq5UBt0YMA][test][0]",
           "searches": [
              {
                 "query": [
                    {
                       "query_type": "BooleanQuery",
                       "lucene": "message:search message:test",
                       "time": "15.52889800ms",
                       "breakdown": {
                          "score": 0,
                          "next_doc": 24495,
                          "match": 0,
                          "create_weight": 8488388,
                          "build_scorer": 7016015,
                          "advance": 0
                       },
                       "children": [
                          {
                             "query_type": "TermQuery",
                             "lucene": "message:search",
                             "time": "4.938855000ms",
                             "breakdown": {
                                "score": 0,
                                "next_doc": 18332,
                                "match": 0,
                                "create_weight": 2945570,
                                "build_scorer": 1974953,
                                "advance": 0
                             }
                          },
                          {
                             "query_type": "TermQuery",
                             "lucene": "message:test",
                             "time": "0.5016660000ms",
                             "breakdown": {
                                "score": 0,
                                "next_doc": 0,
                                "match": 0,
                                "create_weight": 170534,
                                "build_scorer": 331132,
                                "advance": 0
                             }
                          }
                       ]
                    }
                 ],
                 "rewrite_time": 185002,
                 "collector": [
                    {
                       "name": "SimpleTopScoreDocCollector",
                       "reason": "search_top_hits",
                       "time": "2.206529000ms"
                    }
                 ]
              }
           ]
        }
     ]
  }
}
```

- (1) 搜索结果返回，但为了简洁起见，这里省略了

即使是简单的查询，响应也比较复杂。 让我们逐个分解，然后再转到更复杂的例子。

首先，剖析的响应的整体结构如下：

```
{
   "profile": {
        "shards": [
           {
              "id": "[htuC6YnSSSmKFq5UBt0YMA][test][0]",  (1)
              "searches": [
                 {
                    "query": [...],             (2)
                    "rewrite_time": 185002,     (3)
                    "collector": [...]          (4)
                 }
              ]
           }
        ]
     }
}
```

- (1) 为参与响应的每个分片返回一个剖析结果，并由唯一的ID标识
- (2) 每个剖析结果都包含一个包含查询执行细节的部分
- (3) 每个剖析结果都有一个时间表示累积重写时间
- (4) 每个剖析结果还包含一个关于运行搜索的Lucene Collectors的部分

因为可以针对索引中的一个或多个分片执行搜索请求，并且搜索可以覆盖一个或多个索引，所以剖析响应中的顶层元素是shard分片对象的数组。每个分片对象列出了唯一标识分片的id。 ID的格式为[nodeID] [indexName] [shardID]。

Profile本身可以由一个或多个“搜索”组成，其中搜索是针对基本的Lucene索引执行的查询。用户提交的大多数搜索请求只会针对Lucene索引执行单个搜索。但是偶尔会执行多个搜索，例如包括全局聚合（需要为全局上下文执行辅助“match_all”查询）。

在每个search搜索对象内部将有两个剖析信息数组：query查询数组和collector收集器数组。将来可能会添加更多的部分，如suggest，highlight，aggregations等

还将有一个rewrite重写指标，显示重写查询的总时间（以纳秒为单位）。


参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/2.4/search-profile.html

