---
title: "ElasticSearch官网翻译_IndicesRecovery"
date: 2017-06-05 19:18:50
tags: [Elasticsearch]
categories: [Search]
---

## Indices Recovery

Indices recovery API提供观察正在进行恢复的索引分片的功能。 可能针对特定索引或集群范围报告恢复状态。

例如，以下命令将显示索引"index1"和"index2"的恢复信息。

```
GET index1,index2/_recovery?human
```

要查看群集范围的恢复状态，只需省略索引名称。

```
GET /_recovery?human
```

响应：

```
{
  "index1" : {
    "shards" : [ {
      "id" : 0,
      "type" : "SNAPSHOT",
      "stage" : "INDEX",
      "primary" : true,
      "start_time" : "2014-02-24T12:15:59.716",
      "start_time_in_millis": 1393244159716,
      "total_time" : "2.9m",
      "total_time_in_millis" : 175576,
      "source" : {
        "repository" : "my_repository",
        "snapshot" : "my_snapshot",
        "index" : "index1"
      },
      "target" : {
        "id" : "ryqJ5lO5S4-lSFbGntkEkg",
        "hostname" : "my.fqdn",
        "ip" : "10.0.1.7",
        "name" : "my_es_node"
      },
      "index" : {
        "size" : {
          "total" : "75.4mb",
          "total_in_bytes" : 79063092,
          "reused" : "0b",
          "reused_in_bytes" : 0,
          "recovered" : "65.7mb",
          "recovered_in_bytes" : 68891939,
          "percent" : "87.1%"
        },
        "files" : {
          "total" : 73,
          "reused" : 0,
          "recovered" : 69,
          "percent" : "94.5%"
        },
        "total_time" : "0s",
        "total_time_in_millis" : 0
      },
      "translog" : {
        "recovered" : 0,
        "total" : 0,
        "percent" : "100.0%",
        "total_on_start" : 0,
        "total_time" : "0s",
        "total_time_in_millis" : 0,
      },
      "start" : {
        "check_index_time" : "0s",
        "check_index_time_in_millis" : 0,
        "total_time" : "0s",
        "total_time_in_millis" : 0
      }
    } ]
  }
}
```

上述响应显示单个索引恢复单个分片。 在这种情况下，恢复的源是快照存储库，恢复的目标是名称为"my_es_node"的节点。

此外，输出显示恢复的文件的数量和百分比，以及恢复的字节的数量和百分比。

在某些情况下，较高级别的细节可能是优选的。 设置"detailed=true"将显示恢复中的物理文件列表。

```
GET _recovery?human&detailed=true
```

响应：

```
{
  "index1" : {
    "shards" : [ {
      "id" : 0,
      "type" : "STORE",
      "stage" : "DONE",
      "primary" : true,
      "start_time" : "2014-02-24T12:38:06.349",
      "start_time_in_millis" : "1393245486349",
      "stop_time" : "2014-02-24T12:38:08.464",
      "stop_time_in_millis" : "1393245488464",
      "total_time" : "2.1s",
      "total_time_in_millis" : 2115,
      "source" : {
        "id" : "RGMdRc-yQWWKIBM4DGvwqQ",
        "hostname" : "my.fqdn",
        "ip" : "10.0.1.7",
        "name" : "my_es_node"
      },
      "target" : {
        "id" : "RGMdRc-yQWWKIBM4DGvwqQ",
        "hostname" : "my.fqdn",
        "ip" : "10.0.1.7",
        "name" : "my_es_node"
      },
      "index" : {
        "size" : {
          "total" : "24.7mb",
          "total_in_bytes" : 26001617,
          "reused" : "24.7mb",
          "reused_in_bytes" : 26001617,
          "recovered" : "0b",
          "recovered_in_bytes" : 0,
          "percent" : "100.0%"
        },
        "files" : {
          "total" : 26,
          "reused" : 26,
          "recovered" : 0,
          "percent" : "100.0%",
          "details" : [ {
            "name" : "segments.gen",
            "length" : 20,
            "recovered" : 20
          }, {
            "name" : "_0.cfs",
            "length" : 135306,
            "recovered" : 135306
          }, {
            "name" : "segments_2",
            "length" : 251,
            "recovered" : 251
          },
           ...
          ]
        },
        "total_time" : "2ms",
        "total_time_in_millis" : 2
      },
      "translog" : {
        "recovered" : 71,
        "total_time" : "2.0s",
        "total_time_in_millis" : 2025
      },
      "start" : {
        "check_index_time" : 0,
        "total_time" : "88ms",
        "total_time_in_millis" : 88
      }
    } ]
  }
}
```

此响应显示已恢复的实际文件及其大小的详细列表（为简洁起见）。

还显示了各种恢复阶段的时间（以毫秒为单位）：索引检索，translog replay（重新执行更改）和索引开始时间。

请注意，上述列表表示恢复处于"done"阶段。 所有恢复，无论是正在进行还是完成，均保持集群状态，并可随时报告。 设置"active_only=true"将导致仅正在进行的恢复被报告。

以下是完整的选项列表：

值|描述
---|---
detailed|显示详细视图。 这主要用于查看物理索引文件的恢复。 默认值：false。
active_only|仅显示当前正在进行的恢复。 默认值：false。

输出字段说明：

值|描述
---|---
id|分片ID
type|Recovery type（恢复类型）:
 |store（存储）
 |snapshot（快照）
 |replica（副本）
 |relocating（迁移）
stage|Recovery stage（恢复阶段）:
 |init: 恢复还未开始
 |index: 读取索引元数据并复制字节从源到目的地
 |start: 启动引擎，打开索引使用
 |translog: replay（重新执行）事务日志
 |finalize: 清理
 |done: 完成
primary|如果shard是true代表是主分片，否则为false
start_time|恢复开始时间
stop_time|恢复结束时间
total_time_in_millis|以毫秒为单位恢复分片的总时间
source|Recovery source（恢复源）:
 |如果恢复是来自快照的，则为存储库的描述
 |否则为源节点的描述
target|目标节点
index|物理索引恢复统计
translog|事务日志恢复统计
start|打开启动索引时间的统计信息

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/5.4/indices-recovery.html
