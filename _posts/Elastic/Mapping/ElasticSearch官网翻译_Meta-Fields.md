---
title: "ElasticSearch官网翻译_Meta-Fields"
date: 2017-05-12 14:06:34
tags: [Elasticsearch]
categories: [Search]
---

#### Meta-Fields（元字段）

每个文档都有与之关联的元数据，例如_index，映射中的_type和_id元字段。当创建映射类型时，可以自定义一些这些元字段的行为。

##### Identity meta-fields（身份元字段）

- _index : 文档所属的索引
- _uid : 由_type和_id组成的复合字段
- _type : 文档的映射类型
- _id : 文档的ID

##### Document source meta-fields（文档来源元字段）

- _source : 原始的JSON表示文档的正文
- _size : _source字段的大小（以字节为单位），由mapper-size插件提供

##### Indexing meta-fields（索引元字段）

- _all : 一个catch-all（捕获所有）字段，用于索引所有其他字段的值
- _field_names : 文档中包含非空值的所有字段
- _timestamp : 与文档关联的时间戳，手动指定或自动生成
- _ttl : 文档在自动删除之前应该存在多长时间

##### Routing meta-fields（路由元字段）

- _parent : 用于创建两个映射类型之间的parent-child（父子关系）
- _routing : 将文档路由到特定分片的自定义路由值

##### Other meta-field（其他元字段）

- _meta : 应用程序特定元数据

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/2.4/mapping-fields.html
