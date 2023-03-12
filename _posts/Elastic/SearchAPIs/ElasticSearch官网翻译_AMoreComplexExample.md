---
title: "ElasticSearch官网翻译_AMoreComplexExample"
date: 2017-05-28 14:14:44
tags: [Elasticsearch]
categories: [Search]
---

#### A more complex example

为了演示一个稍微更复杂的查询和关联的结果，我们可以对以下查询进行配置：

```
GET /test/_search
{
  "profile": true,
  "query": {
    "term": {
      "message": {
        "value": "search"
      }
    }
  },
  "aggs": {
    "non_global_term": {
      "terms": {
        "field": "agg"
      },
      "aggs": {
        "second_term": {
          "terms": {
            "field": "sub_agg"
          }
        }
      }
    },
    "another_agg": {
      "cardinality": {
        "field": "aggB"
      }
    },
    "global_agg": {
      "global": {},
      "aggs": {
        "my_agg2": {
          "terms": {
            "field": "globalAgg"
          }
        }
      }
    }
  },
  "post_filter": {
    "term": {
      "my_field": "foo"
    }
  }
}
```

这个例子有：

- A query
- A scoped aggregation
- A global aggregation
- A post_filter

响应为：

```
{
   "profile": {
         "shards": [
            {
               "id": "[P6-vulHtQRWuD4YnubWb7A][test][0]",
               "searches": [
                  {
                     "query": [
                        {
                           "query_type": "TermQuery",
                           "lucene": "my_field:foo",
                           "time": "0.4094560000ms",
                           "breakdown": {
                              "score": 0,
                              "next_doc": 0,
                              "match": 0,
                              "create_weight": 31584,
                              "build_scorer": 377872,
                              "advance": 0
                           }
                        },
                        {
                           "query_type": "TermQuery",
                           "lucene": "message:search",
                           "time": "0.3037020000ms",
                           "breakdown": {
                              "score": 0,
                              "next_doc": 5936,
                              "match": 0,
                              "create_weight": 185215,
                              "build_scorer": 112551,
                              "advance": 0
                           }
                        }
                     ],
                     "rewrite_time": 7208,
                     "collector": [
                        {
                           "name": "MultiCollector",
                           "reason": "search_multi",
                           "time": "1.378943000ms",
                           "children": [
                              {
                                 "name": "FilteredCollector",
                                 "reason": "search_post_filter",
                                 "time": "0.4036590000ms",
                                 "children": [
                                    {
                                       "name": "SimpleTopScoreDocCollector",
                                       "reason": "search_top_hits",
                                       "time": "0.006391000000ms"
                                    }
                                 ]
                              },
                              {
                                 "name": "BucketCollector: [[non_global_term, another_agg]]",
                                 "reason": "aggregation",
                                 "time": "0.9546020000ms"
                              }
                           ]
                        }
                     ]
                  },
                  {
                     "query": [
                        {
                           "query_type": "MatchAllDocsQuery",
                           "lucene": "*:*",
                           "time": "0.04829300000ms",
                           "breakdown": {
                              "score": 0,
                              "next_doc": 3672,
                              "match": 0,
                              "create_weight": 6311,
                              "build_scorer": 38310,
                              "advance": 0
                           }
                        }
                     ],
                     "rewrite_time": 1067,
                     "collector": [
                        {
                           "name": "GlobalAggregator: [global_agg]",
                           "reason": "aggregation_global",
                           "time": "0.1226310000ms"
                        }
                     ]
                  }
               ]
            }
         ]
      }
}
```

你可以看到，从前面的输出显然是冗长的。 查询的所有主要部分都表示：

1. 第一个TermQuery（message:search）表示主要term查询
2. 第二个TermQuery（my_field:foo）表示post_filter查询
3. 有一个MatchAllDocsQuery（*:*）查询被执行为第二个不同的搜索。 这不是用户指定的查询的一部分，而是由全局聚合自动生成，以提供全局查询范围

收集器树是相当简单的，显示单个MultiCollector如何包装一个FilteredCollector来执行post_filter（并反过来包装正常的得分SimpleCollector），一个BucketCollector来运行所有作用域聚合。 在MatchAll搜索中，有一个GlobalAggregator可以运行全局聚合。

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/2.4/_a_more_complex_example.html

