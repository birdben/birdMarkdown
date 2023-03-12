---
title: "ElasticSearch官网翻译_Preference"
date: 2017-05-26 01:09:05
tags: [Elasticsearch]
categories: [Search]
---

#### Preference（首选项）

控制哪些分片副本的preference（首选项）执行搜索请求。 默认情况下，操作在分片副本之间随机化。

preference是一个query string（查询字符串）参数，可以设置为：

值|描述
---|---
_primary|该操作将仅在主分片上执行。
_primary_first|该操作将在主分片上执行，如果不可用（故障转移），则将在其他分片上执行。
_replica|该操作将仅在副本分片上执行。
_replica_first|该操作将仅在副本分片上执行，如果不可用（故障切换），则将在其他分片上执行。
_local|如果可能的话，该操作将更倾向于在本地分配的分片上执行。
_only_node:xyz|限制搜索仅在提供的节点id（当前例子为xyz）的节点上执行。
_prefer_node:xyz|如果适用，则在提供的节点id（当前例子为xyz）的节点上执行。
_shards:2,3|限制操作到指定的分片。 （当前例子为2和3）。 此偏好可以与其他偏好相结合，但必须首先显示：_shards:2,3;_primary
_only_nodes|将操作限制在https://www.elastic.co/guide/en/elasticsearch/reference/current/cluster.html节点规范中指定的节点
Custom (string) value|将使用自定义值来确保相同的分片将用于相同的自定义值。 这可以帮助在不同刷新状态下命中不同分片时的“跳跃值”。 示例值可以与Web会话ID或用户名类似。

例如，使用用户的会话ID来确保用户对结果的一致排序：

```
curl localhost:9200/_search?preference=xyzabc123 -d '
{
    "query": {
        "match": {
            "title": "elasticsearch"
        }
    }
}
'
```

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/2.4/search-request-preference.html

