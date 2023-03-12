---
title: "ElasticSearch官网翻译_CreateIndex"
date: 2017-06-03 14:28:15
tags: [Elasticsearch]
categories: [Search]
---

## Create Index

Create Index API允许实例化索引。 Elasticsearch提供对多个索引的支持，包括跨多个索引执行操作。

### Index Settings（索引设置）

创建的每个索引可以具有与之相关联的特定设置。

```
PUT twitter
{
    "settings" : {
        "index" : {
            "number_of_shards" : 3, (1)
            "number_of_replicas" : 2 (2)
        }
    }
}
```

- (1) number_of_shards的默认值为5
- (2) number_of_replicas的默认值为1（即每个主分片有一个副本）

上面的第二curl示例显示了如何使用YAML为其创建具有特定设置的名为twitter的索引。 在这种情况下，创建一个具有3个分片的索引，每个分片具有2个副本。 索引设置也可以用JSON定义：

```
PUT twitter
{
    "settings" : {
        "index" : {
            "number_of_shards" : 3,
            "number_of_replicas" : 2
        }
    }
}
```

或更简化

```
PUT twitter
{
    "settings" : {
        "number_of_shards" : 3,
        "number_of_replicas" : 2
    }
}
```

###### 注意

> 你不必在settings部分中显式指定index部分。

当创建索引时可以设置的所有不同索引级别设置的更多相关信息，请检查索引模块部分。

### Mappings（映射）

create index API允许提供一组一个或多个mappings映射：

```
PUT test
{
    "settings" : {
        "number_of_shards" : 1
    },
    "mappings" : {
        "type1" : {
            "properties" : {
                "field1" : { "type" : "text" }
            }
        }
    }
}
```

### Aliases（别名）

create index API还允许提供一组aliases别名：

```
PUT test
{
    "aliases" : {
        "alias_1" : {},
        "alias_2" : {
            "filter" : {
                "term" : {"user" : "kimchy" }
            },
            "routing" : "kimchy"
        }
    }
}
```

### Wait For Active Shards（等待活动的分片）

默认情况下，创建索引时，只有当每个分片的主副本已经启动或请求超时时，才会向客户端返回响应。 索引创建响应将指示发生了什么：

```
{
    "acknowledged": true,
    "shards_acknowledged": true
}
```

acknowledged指示是否在集群中成功创建了索引，而shards_acknowledged指示是否在超时之前为索引中的每个分片启动必要数量的分片副本。请注意，acknowledged或shards_acknowledledged仍然可能是false，但索引创建成功。这些值只是表示操作是否在超时前完成。如果acknowledged为false，则在使用新创建的索引更新群集状态之前我们超时了，但很可能会在某个时间创建。如果shards_acknowledledged为false，那么在必须数量的分片启动（默认情况下只是主分片）之前，我们超时了，即使集群状态已成功更新，以显示新创建的索引（即acknowledged=true）。

我们可以通过索引设置index.write.wait_for_active_shards来更改仅等待主分片的默认值（请注意，更改此设置也会影响所有后续写入操作的wait_for_active_shards值）：

```
PUT test
{
    "settings": {
        "index.write.wait_for_active_shards": "2"
    }
}
```

或通过请求参数wait_for_active_shards：

```
PUT test?wait_for_active_shards=2
```

有关wait_for_active_shards及其可接受值的详细说明，请参见此处。

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/5.4/indices-create-index.html
