---
title: "ElasticSearch官网翻译_Rescoring"
date: 2017-05-25 15:18:47
tags: [Elasticsearch]
categories: [Search]
---

#### Rescoring（二次计算得分）

<b>
重新计算可以通过使用次要（通常更昂贵的）算法来重新排序query和post_filter阶段重新排序返回的顶部（例如100 - 500）文档，而不是将昂贵的算法应用于索引中的所有文档来帮助提高精度。

在返回其结果以由处理整个搜索请求的节点排序之前，在每个分片上执行rescore请求。
</b>

目前，rescore API只有一个实现：查询rescorer，它使用查询来调整评分。 在将来，可以提供可选择的rescorer，例如，成对的rescorer。

###### 注意

<b>当使用sort或search_type设置为scan或count时，rescore阶段不执行。</b>

###### 注意

<b>当你向用户显示分页时，你不应该在逐步浏览每个页面（通过传递不同的from值）时更改window_size，因为这可以改变顶部命中，导致结果在用户逐步浏览页面时变得混乱。</b>

##### Query rescorer

查询rescorer仅对查询和post_filter阶段返回的Top-K结果执行第二个查询。 将在每个分片上检查的文档数量可以通过window_size参数控制，该参数默认为from和size。

<b>
默认情况下，原始查询和rescore查询的分数线性组合，以生成每个文档的最终_score。 可以使用query_weight和rescore_query_weight控制原始查询和rescore查询的相对重要性。 两者默认为1。
</b>

例如：

```
curl -s -XPOST 'localhost:9200/_search' -d '{
   "query" : {
      "match" : {
         "field1" : {
            "operator" : "or",
            "query" : "the quick brown",
            "type" : "boolean"
         }
      }
   },
   "rescore" : {
      "window_size" : 50,
      "query" : {
         "rescore_query" : {
            "match" : {
               "field1" : {
                  "query" : "the quick brown",
                  "type" : "phrase",
                  "slop" : 2
               }
            }
         },
         "query_weight" : 0.7,
         "rescore_query_weight" : 1.2
      }
   }
}
'
```

分数组合的方式可以通过score_mode来控制：

Score Mode|Description
---|---
total|将原始分数和rescore查询分数相加。 默认值。
multiply|将原始分数和rescore查询分数相乘。 用于函数查询分配。
avg|取原始得分和rescore查询得分的平均值。
max|取原始分数和rescore查询分数的最大值。
min|取原始分数和rescore查询分数的最小值。

##### Multiple Rescores

也可以按顺序执行多个rescore查询得分：

```
curl -s -XPOST 'localhost:9200/_search' -d '{
   "query" : {
      "match" : {
         "field1" : {
            "operator" : "or",
            "query" : "the quick brown",
            "type" : "boolean"
         }
      }
   },
   "rescore" : [ {
      "window_size" : 100,
      "query" : {
         "rescore_query" : {
            "match" : {
               "field1" : {
                  "query" : "the quick brown",
                  "type" : "phrase",
                  "slop" : 2
               }
            }
         },
         "query_weight" : 0.7,
         "rescore_query_weight" : 1.2
      }
   }, {
      "window_size" : 10,
      "query" : {
         "score_mode": "multiply",
         "rescore_query" : {
            "function_score" : {
               "script_score": {
                  "script": "log10(doc['numeric'].value + 2)"
               }
            }
         }
      }
   } ]
}
'
```

<b>
第一个获取查询的结果，第二个获取第一个结果，等等。第二个rescore将“查看”第一个rescore进行的排序，因此可以在第一个rescore上使用一个大窗口将文档拉入较小的窗口中进行第二次重新排序。
</b>

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/2.4/search-request-rescore.html

