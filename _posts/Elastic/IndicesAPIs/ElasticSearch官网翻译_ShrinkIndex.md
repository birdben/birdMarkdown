---
title: "ElasticSearch官网翻译_ShrinkIndex"
date: 2017-06-03 15:26:16
tags: [Elasticsearch]
categories: [Search]
---

## Shrink Index

Shrink API允许你将现有索引缩小到具有较少主分片的新索引。目标索引中所请求的主分片数量必须是源索引中分片数量的一个因子。例如，一个具有8个主分片的索引可以缩小为4个，2个或1个主分片，或者一个具有15个主分片的索引可以缩小为5个，3个或1个主分片。如果索引中的分片数量是素数，则可以只能缩小成单个的主分片。收缩之前，索引中每个分片的副本（主分片或副本分片）必须存在于同一个节点上。

收缩工作如下：

- 首先，它创建与源索引具有相同定义的新的目标索引，但是具有较小数量的主分片。
- 然后，将来自源索引的段hard-links硬链接到目标索引中。（如果文件系统不支持硬链接，则所有段都将复制到新索引中，这是一个更耗时的过程。）
- 最后，它恢复了目标索引，好像它是一个关闭的索引刚被重新打开一样。

### Preparing an index for shrinking（准备收缩索引）

为了缩小索引，索引必须标记为只读，并且索引中每个分片的副本（主分片和副本分片）必须重定位到同一节点并具有health运行状况为green。

这两个条件可以通过以下请求实现：

```
PUT /my_source_index/_settings
{
  "settings": {
    "index.routing.allocation.require._name": "shrink_node_name", (1)
    "index.blocks.write": true (2)
  }
}
```

- 强制将每个分片的副本重定位到名称为shrink_node_name的节点。 有关更多选项，请参阅Shard Allocation Filtering。
- 防止对此索引的写入操作，同时仍允许元数据更改，如删除索引。

重新分配源索引可能需要一段时间。 可以使用_cat recovery API跟踪进度，或者可以使用cluster health API通过设置wait_for_no_relocating_shards参数等待直到所有分片重新分配完成。

### Shrinking an index（收缩索引）

要将my_source_index收缩为一个名为my_target_index的新索引，请发出以下请求：

```
POST my_source_index/_shrink/my_target_index
```

一旦将目标索引被添加到集群状态，上述请求将立即返回 - 它不等待收缩操作启动。

###### 重要

> 如果符合以下要求，索引只能缩小：
> 
> - 目标索引必须不存在
> 
> - 索引必须比目标索引更多的主分片。
> 
> - 目标索引中的主分片数量必须是源索引中主分片数的一个因子。 源索引必须具有比目标索引更多的主分片。
> 
> - 该索引不能包含跨所有分片超过2,147,483,519的总文档数，这些文档将缩小到目标索引中的单个分片中，因为这是可以适合单个分片的文档的最大数量。
> 
> - 处理收缩过程的节点必须具有足够的可用磁盘空间以容纳现有索引的第二个副本。

_shrink API类似于create index API，并接受目标索引的settings和aliases参数：

```
POST my_source_index/_shrink/my_target_index
{
  "settings": {
    "index.number_of_replicas": 1,
    "index.number_of_shards": 1, (1)
    "index.codec": "best_compression" (2)
  },
  "aliases": {
    "my_search_indices": {}
  }
}
```

- (1) 目标索引中的分片数。 这必须是源索引中分片数量的一个因子。
- (2) best_compression只有在对索引进行新的写入时才会起作用，例如强制将分片合并到单个段时。

###### 注意

> _shrink请求中可能未指定映射，并且所有index.analysis.*和index.similarity.*设置将被来自源索引的设置覆盖。

### Monitoring the shrink process（监控收缩进程）

可以使用_cat recovery API监视收缩过程，或者通过将wait_for_status参数设置为yellow，可以使用cluster health API等待所有主分片被分配完成。

当目标索引被添加到集群状态之后，_shrink API将尽快返回，在任何分片被分配之前。 在这一刻，所有的分片都处于unassigned未分配的状态。 如果由于任何原因，目标索引无法在收缩节点上分配，则其主分片将保持unassigned未分配状态，直到它可以在该节点上分配。

一旦分配了主分片，它将转到initializing初始化状态，并且收缩过程开始。 收缩操作完成后，分片将变为active活动状态。 在这一刻，Elasticsearch将尝试分配任何副本，并可能决定将主分片重分配到另一个节点。

### Wait For Active Shards（等待活动的分片）

因为收缩操作会创建一个新的索引来收缩分片，所以在创建索引时wait_for_active_shards参数设置也适用于shrink index操作。

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/5.4/indices-shrink-index.html
