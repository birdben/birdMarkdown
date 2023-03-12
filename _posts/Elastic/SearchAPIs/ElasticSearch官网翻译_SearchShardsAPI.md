---
title: "ElasticSearch官网翻译_SearchShardsAPI"
date: 2017-05-27 01:39:23
tags: [Elasticsearch]
categories: [Search]
---

#### Search Shards API

Search shards api返回要执行搜索请求的索引和分片。 这可以为处理问题或通过路由选择和分片优先级进行规划优化提供有用的反馈。

index和type参数可以是单个值，也可以是逗号分隔。

##### 用法

完整例子：

```
curl -XGET 'localhost:9200/twitter/_search_shards'
```

这将产生以下结果：

```
{
  "nodes": {
    "JklnKbD7Tyqi9TP3_Q_tBg": {
      "name": "Rl'nnd",
      "transport_address": "inet[/192.168.1.113:9300]"
    }
  },
  "shards": [
    [
      {
        "index": "twitter",
        "node": "JklnKbD7Tyqi9TP3_Q_tBg",
        "primary": true,
        "relocating_node": null,
        "shard": 3,
        "state": "STARTED"
      }
    ],
    [
      {
        "index": "twitter",
        "node": "JklnKbD7Tyqi9TP3_Q_tBg",
        "primary": true,
        "relocating_node": null,
        "shard": 4,
        "state": "STARTED"
      }
    ],
    [
      {
        "index": "twitter",
        "node": "JklnKbD7Tyqi9TP3_Q_tBg",
        "primary": true,
        "relocating_node": null,
        "shard": 0,
        "state": "STARTED"
      }
    ],
    [
      {
        "index": "twitter",
        "node": "JklnKbD7Tyqi9TP3_Q_tBg",
        "primary": true,
        "relocating_node": null,
        "shard": 2,
        "state": "STARTED"
      }
    ],
    [
      {
        "index": "twitter",
        "node": "JklnKbD7Tyqi9TP3_Q_tBg",
        "primary": true,
        "relocating_node": null,
        "shard": 1,
        "state": "STARTED"
      }
    ]
  ]
}
```

并指定相同的请求，这次使用路由值：

```
curl -XGET 'localhost:9200/twitter/_search_shards?routing=foo,baz'
```

这将产生以下结果：

```
{
  "nodes": {
    "JklnKbD7Tyqi9TP3_Q_tBg": {
      "name": "Rl'nnd",
      "transport_address": "inet[/192.168.1.113:9300]"
    }
  },
  "shards": [
    [
      {
        "index": "twitter",
        "node": "JklnKbD7Tyqi9TP3_Q_tBg",
        "primary": true,
        "relocating_node": null,
        "shard": 2,
        "state": "STARTED"
      }
    ],
    [
      {
        "index": "twitter",
        "node": "JklnKbD7Tyqi9TP3_Q_tBg",
        "primary": true,
        "relocating_node": null,
        "shard": 4,
        "state": "STARTED"
      }
    ]
  ]
}
```

由于路由值已经被指定，所以这次搜索只能针对两个分片执行。

##### 所有参数

值|描述
---|---
routing|以逗号分隔的路由值的列表来确定应该执行请求的分片。
preference|控制哪些分片副本的preference来执行搜索请求。 默认情况下，操作在分片副本之间随机化。 有关所有可接受值的列表，请参阅偏好文档。
local|一个布尔值，用来设置是否在本地读取集群状态以确定哪些分片被分配，而不是使用主节点的集群状态。

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/2.4/search-shards.html

