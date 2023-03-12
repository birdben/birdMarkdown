---
title: "ElasticSearch官网翻译_ShadowReplicaIndices"
date: 2017-06-05 15:56:36
tags: [Elasticsearch]
categories: [Search]
---

## Shadow replica indices

###### 警告

> 在5.2.0中弃用。
Shadow replicas（影子副本）没有看到太多的用法，我们打算删除它们

如果要使用共享文件系统，可以使用影子副本设置来选择要保留索引的数据的位置，以及Elasticsearch应如何在索引的所有副本分片上重放操作。

为了充分利用index.data_path和index.shadow_replicas设置，你需要允许Elasticsearch通过在elasticsearch.yml中将node.add_lock_id_to_custom_path设置为false来为多个实例使用相同的数据目录：

```
node.add_lock_id_to_custom_path: false
```

你还需要向安全管理员指明自定义索引的位置，以便可以应用正确的权限。 你可以通过在elasticsearch.yml中设置path.shared_data设置来执行此操作：

```
path.shared_data: /opt/data
```

这意味着Elasticsearch可以读取和写入path.shared_data设置的任何子目录中的文件。

然后，你可以使用自定义数据路径创建索引，其中每个节点将使用此路径的数据：

###### 警告

> 因为Shadow replicas（影子副本）不对副本分片上的文档进行索引，如果没有在包含副本的节点上处理最新的集群状态，则副本的已知映射可能位于索引的已知映射之后。 因此，强烈建议你在使用Shadow replicas（影子副本）时使用预定义的映射。

```
PUT /my_index
{
    "index" : {
        "number_of_shards" : 1,
        "number_of_replicas" : 4,
        "data_path": "/opt/data/my_index",
        "shadow_replicas": true
    }
}
```

###### 警告

> 在上述示例中，"/opt/data/my_index"路径是一个共享文件系统，必须在Elasticsearch集群中的每个节点上可用。 你还必须确保Elasticsearch进程具有读取和写入index.data_path设置中使用的目录的正确权限。

data_path不必包含索引名称，在这个例子中，使用"my_index"，但它也可以设置为"/opt/data/"

已将index.shadow_replicas设置设置为"true"创建的索引将不会将文档操作复制到任何副本分片，而只会不断刷新。 一旦段可用于影子副本所在的文件系统（在Elasticsearch"flush"之后），可以使用常规刷新（由index.refresh_interval控制）来使新数据可搜索。

###### 注意

> 由于文档仅在主分片上进行索引，所以如果在副本分片上执行，实时GET请求可能无法返回文档，因此，如果没有设置preference flag，GET API请求将自动设置"?preference=_primary"标志。

为了确保数据以足够快的速度同步，你可能需要将索引的flush阈值调整为所需数值。 flush需要fsync段文件到磁盘，所以它们将对所有其他副本节点可见。 用户应该测试他们合适的flush阈值水平，因为flushing可能会影响索引性能。

Elasticsearch集群仍将检测到主分片的丢失，并在此情况下将副本分片转换为主分片。 因为没有为每个Shadow replicas（影子副本）维护IndexWriter，所以此转换将需要稍长时间。

以下是可以使用update settings API更改的settings设置列表：

- index.data_path (string)

用于索引数据的路径。 请注意，默认情况下，Elasticsearch会将节点序号附加到路径，以确保同一台机器上的多个Elasticsearch实例不共享同一个数据目录。

- index.shadow_replicas

指示此索引的布尔值应使用Shadow replicas（影子副本）。 默认为false。

- index.shared_filesystem

指示此索引的布尔值使用共享文件系统。 如果index.shadow_replicas设置为true，则默认为true，否则为false。

- index.shared_filesystem.recover_on_any_node

指示是否允许索引的主分片在集群中的任何节点上恢复的布尔值。 如果找到保存分片副本的节点，则恢复优先于该节点。 默认为false。

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/5.4/indices-shadow-replicas.html
