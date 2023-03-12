---
title: "ElasticSearch官网翻译_IndicesShardStores"
date: 2017-06-05 19:51:23
tags: [Elasticsearch]
categories: [Search]
---

## Indices Shard Stores

提供索引分片副本的存储信息。 存储信息报告有哪些节点存在分片副本，分片副本分配的ID，每个分片副本的唯一标识符以及当打开分片索引时或更早之前引擎故障遇到的任何异常。

默认情况下，只列出至少有一个unallocated copy（未分配副本）的分片的存储信息。 当群集运行状况为黄色时，这将列出至少有一个unassigned replica（未分配副本）的分片的存储信息。 当群集健康状态为红色时，这将列出分片的存储信息，该分片具有未分配的主分片。

Endpoints包括特定索引，几个索引或全部的分片存储信息：

```
curl -XGET 'http://localhost:9200/test/_shard_stores'
curl -XGET 'http://localhost:9200/test1,test2/_shard_stores'
curl -XGET 'http://localhost:9200/_shard_stores'
```

可以通过status参数来更改列出存储信息的分片范围。 默认为黄色和红色。 黄色列出存储具有至少一个unassigned replica（未分配副本）的分片信息，红色用于未分配的主分片的分片。 使用绿色列出具有所有已分配副本的分片的存储信息。

```
curl -XGET 'http://localhost:9200/_shard_stores?status=green'
```

响应：

分片存储信息按索引和分片ids分组。

```
{
    ...
   "0": { (1)
        "stores": [ (2)
            {
                "sPa3OgxLSYGvQ4oPs-Tajw": { (3)
                    "name": "node_t0",
                    "transport_address": "local[1]",
                    "attributes": {
                        "mode": "local"
                    }
                },
                "allocation_id": "2iNySv_OQVePRX-yaRH_lQ", (4)
                "legacy_version": 42, (5)
                "allocation" : "primary" | "replica" | "unused", (6)
                "store_exception": ... (7)
            },
            ...
        ]
   },
    ...
}
```

- (1) key是存储信息的相应分片ID
- (2) 分片所有副本的存储信息列表
- (3) 托管存储副本的节点信息，key是唯一节点ID。
- (4) 存储副本的分配ID
- (5) 存储副本的版本（仅适用于当前版本的Elasticsearch中不再适用的旧版分片副本）
- (6) 存储副本的状态，无论是用作primary，replica还是unused
- (7) 打开分片索引时或更早之前引擎故障遇到的任何异常

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/5.4/indices-shards-stores.html
