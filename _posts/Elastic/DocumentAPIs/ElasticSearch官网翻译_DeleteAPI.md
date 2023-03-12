---
title: "ElasticSearch官网翻译_DeleteAPI"
date: 2017-05-31 14:28:41
tags: [Elasticsearch]
categories: [Search]
---

## Delete API

Delete API允许从特定索引中根据其id删除一个类型的JSON文档。 以下示例从名为twitter的索引中删除JSON文档，类型名为tweet，其值为1：

```
$ curl -XDELETE 'http://localhost:9200/twitter/tweet/1'
```

上述删除操作的结果是：

```
{
    "_shards" : {
        "total" : 10,
        "failed" : 0,
        "successful" : 10
    },
    "found" : true,
    "_index" : "twitter",
    "_type" : "tweet",
    "_id" : "1",
    "_version" : 2,
    "result": "deleted"
}
```

### Versioning（版本控制）

索引的每个文档都被版本化。 删除文档时，可以指定version，以确保我们正在尝试删除的相关文档实际上被删除，同时没有更改。 对文档执行的每个写入操作，包含删除操作，都使其版本号增加。

### Routing（路由）

当使用控制routing路由的能力进行索引时，为了删除文档，还应提供routing路由值。 例如：

```
$ curl -XDELETE 'http://localhost:9200/twitter/tweet/1?routing=kimchy'
```

以上将删除id为1的tweet，但将根据用户进行routing路由。 注意，发出的删除请求没有正确的routing路由，将导致文档不能被删除。

当_routing映射被设置为required，并且没有指定routing路由值时，delete api将抛出RoutingMissingException异常并拒绝该请求。

### Parent（父级）

可以设置parent参数，这与设置routing路由参数基本相同。

请注意，删除父文档不会自动删除其子文档。 删除给定父文档ID的所有子文档的一种方法是，使用Delete By Query API执行带有自动生成（和索引）字段_parent的索引，它的格式为parent_type#parent_id。

当删除子文档时，必须指定其父ID，否则删除请求将被拒绝，并抛出RoutingMissingException异常。

### Automatic index creation（自动创建索引）

如果以前没有创建索引，则删除操作会自动创建一个索引（查看用于手动创建索引的create index API），并且如果之前尚未创建（类型），会根据指定的类型名与动态映射类型来自动创建类型（查看用于手动创建类型映射的put mapping API）。

### Distributed（分布式）

删除操作将被哈希化为特定的分片ID。 然后将其重定向到该分片副本ID组中的主分片，并将其复制（如果需要）到该分片副本ID组中的副本分片。

### Wait For Active Shards（等待活动分片）

执行删除请求时，可以设置wait_for_active_shards参数，以便在开始处理删除请求之前（检查）要求最小数量的分片副本处于活动状态。 有关详细信息和使用示例，请参阅此处。

### Refresh（刷新）

控制此请求所做的更改对于搜索是可见的。 参与?refresh。

### Timeout（超时）

分配执行删除操作的主分片在执行删除操作时可能不可用。 一些原因可能是主分片目前正在从持久化存储中恢复或正在进行重新定位。 默认情况下，删除操作将等待主分片可用最多1分钟，然后才发生故障并响应错误。 timeout参数可用于显式指定等待多长时间。 以下是将其设置为5分钟的示例：

```
$ curl -XDELETE 'http://localhost:9200/twitter/tweet/1?timeout=5m'
```

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/5.4/docs-delete.html
