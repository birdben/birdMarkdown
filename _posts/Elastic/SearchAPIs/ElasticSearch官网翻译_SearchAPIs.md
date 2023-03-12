---
title: "ElasticSearch官网翻译_SearchAPIs"
date: 2017-05-23 16:08:13
tags: [Elasticsearch]
categories: [Search]
---

#### Search APIs

大多数搜索API是multi-index，multi-type，除了Explain API端点。

##### Routing（路由）

当执行搜索时，它将被广播到所有index（索引）/indices shards（索引碎片）（复制之间的循环）。 可以通过提供routing参数来控制哪些shared分片将被搜索。 例如，索引tweet时，routing值可以是用户名：

```
$ curl -XPOST 'http://localhost:9200/twitter/tweet?routing=kimchy' -d '{
    "user" : "kimchy",
    "postDate" : "2009-11-15T14:12:12",
    "message" : "trying out Elasticsearch"
}'
```

在这种情况下，如果要仅搜索特定user的tweets，我们可以将其指定为routing，导致搜索仅触发相关的分片：

```
$ curl -XGET 'http://localhost:9200/twitter/tweet/_search?routing=kimchy' -d '{
    "query": {
        "bool" : {
            "must" : {
                "query_string" : {
                    "query" : "some query string here"
                }
            },
            "filter" : {
                "term" : { "user" : "kimchy" }
            }
        }
    }
}'
```

路由参数可以是多值，以逗号分隔的字符串表示。 这将导致命中路由值匹配的相关分片。

##### Stats Group（统计组）

搜索可以与统计组相关联，维护每个组的统计信息聚合。 稍后可以使用indices stats API进行取回。 例如，这是一个搜索请求主体，将请求与两个不同的组相关联：

```
{
    "query" : {
        "match_all" : {}
    },
    "stats" : ["group1", "group2"]
}
```

##### Global Search Timeout（全局搜索超时）

单独搜索可以将timeout作为Request Body Search的一部分。 由于搜索请求可能来自许多来源，Elasticsearch具有全局搜索超时的动态群集级别设置，适用于在Request Body Search中未设置timeout超时的所有搜索请求。 默认值没有全局超时。 设置键为search.default_search_timeout，可以使用Cluster Update Settings端点进行设置。 将此值设置为-1会将全局搜索超时重置为无超时。

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/2.4/search.html
