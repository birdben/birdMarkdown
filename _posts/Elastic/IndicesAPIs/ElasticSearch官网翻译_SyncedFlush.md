---
title: "ElasticSearch官网翻译_SyncedFlush"
date: 2017-06-06 00:26:23
tags: [Elasticsearch]
categories: [Search]
---

## Synced Flush

Elasticsearch跟踪每个分片的索引活动。尚未收到任何索引操作5分钟的分片将自动标记为不活动。这为Elasticsearch提供了一个减少分片资源的机会，并且还可以执行一种称为synced flush（同步刷新）的特殊类型的flush。synced flush（同步刷新）将执行正常flush（刷新），然后将生成的唯一标记（sync_id）添加到所有分片。

由于当没有进行索引操作时添加了同步标识符，因此可以将其用作检查两个分片的lucene索引是否相同的快速方法。这种快速sync id比较（如果存在）在恢复或重新启动期间被使用以跳过过程的第一个也是最昂贵的阶段。在这种情况下，不需要复制段文件，并且恢复的事务日志replay（重新执行）阶段可以立即启动。请注意，由于sync id标记与flush一起应用，因此transaction log（事务日志）很有可能是空的，从而加快了恢复的速度。

这对于具有很多从未或很少更新的索引（例如基于时间的数据）的用例特别有用。这种用例通常产生大量的索引，其恢复没有sync flush marker（同步的清除标记）将需要很长时间。

要检查分片是否具有标记，请查找indices stats API返回的分片统计信息的commit部分：

```
GET twitter/_stats?level=shards
```

它返回类似于以下内容：

```
{
   ...
   "indices": {
      "twitter": {
         "primaries": {},
         "total": {},
         "shards": {
            "0": [
               {
                  "routing": {
                     ...
                  },
                  "commit": {
                     "id": "te7zF7C4UsirqvL6jp/vUg==",
                     "generation": 2,
                     "user_data": {
                        "sync_id": "AU2VU0meX-VX2aNbEUsD" ,(1)
                        ...
                     },
                     "num_docs": 0
                  }
               }
               ...
            ],
            ...
         }
      }
   }
}
```

- (1) sync id标记

### Synced Flush API（同步刷新API）

Synced Flush API允许管理员手动启动同步刷新。这对于计划（滚动）群集重新启动特别有用，你可以在其中停止索引，并且不想等待默认的5分钟空闲索引自动sync-flushed（同步刷新）。

虽然方便，但有一些关于此API的注意事项：

1. Synced flush（同步刷新）是一个尽力而为的操作。任何正在进行的索引操作将导致synced flush（同步刷新）在该分片上失败。这意味着有些分片可能被synced flush（同步刷新），而其他分片可能没有被synced flush（同步刷新）。详见下文。
2. 一旦分片再次刷新后，sync_id标记将被删除。这是因为flush（刷新）代替存储标记的低级别lucene提交点。事务日志中未提交的操作不会删除标记。在实践中，应该考虑在索引上的任何索引操作，因为在任何时候可以由Elasticsearch触发删除标记的flush（刷新）。

###### 注意

> 在进行索引时请求synced flush（同步刷新）是无害的。空闲的分片会成功，分片不会失败。任何成功的分片将具有更快的恢复时间。

```
POST twitter/_flush/synced
```

响应包含有关成功sync-flushed（同步刷新）多个分片的信息以及有关任何故障的信息。

当两个分片和一个副本索引的所有分片成功sync-flushed（同步刷新）时，这是什么样子：

```
{
   "_shards": {
      "total": 2,
      "successful": 2,
      "failed": 0
   },
   "twitter": {
      "total": 2,
      "successful": 2,
      "failed": 0
   }
}
```

以下是当一个分片组由于pending操作而失败时看起来像：

```
{
   "_shards": {
      "total": 4,
      "successful": 2,
      "failed": 2
   },
   "twitter": {
      "total": 4,
      "successful": 2,
      "failed": 2,
      "failures": [
         {
            "shard": 1,
            "reason": "[2] ongoing operations on primary"
         }
      ]
   }
}
```

###### 注意

> 当由于并发索引操作而导致synced flush（同步刷新）失败时，会显示上述错误。 在这种情况下，HTTP状态代码将为409 CONFLICT。

有时这些故障是专用于分片副本的。 失败的副本将不符合快速恢复资格，但成功的副本仍将是。 此例子报告如下：

```
{
   "_shards": {
      "total": 4,
      "successful": 1,
      "failed": 1
   },
   "twitter": {
      "total": 4,
      "successful": 3,
      "failed": 1,
      "failures": [
         {
            "shard": 1,
            "reason": "unexpected error",
            "routing": {
               "state": "STARTED",
               "primary": false,
               "node": "SZNr2J_ORxKTLUCydGX4zA",
               "relocating_node": null,
               "shard": 1,
               "index": "twitter"
            }
         }
      ]
   }
}
```

###### 注意

> 当分片复制无法sync-flush（同步刷新）时，返回的HTTP状态代码将为409 CONFLICT。

synced flush API可以通过单个调用应用于多个索引，甚至可以应用于_all的所有索引。

```
POST kimchy,elasticsearch/_flush/synced

POST _flush/synced
```

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/5.4/indices-synced-flush.html
