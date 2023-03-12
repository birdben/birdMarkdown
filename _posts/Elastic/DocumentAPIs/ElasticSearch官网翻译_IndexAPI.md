---
title: "ElasticSearch官网翻译_IndexAPI"
date: 2017-05-30 17:30:30
tags: [Elasticsearch]
categories: [Search]
---

## Index API

Index API在特定索引中添加或更新一个类型的JSON文档，使其可搜索。 以下示例将JSON文档插入到"twitter"索引中，名称为"tweet"的类型，ID为1：

```
PUT twitter/tweet/1
{
    "user" : "kimchy",
    "post_date" : "2009-11-15T14:12:12",
    "message" : "trying out Elasticsearch"
}
```

上述索引操作的结果是：

```
{
    "_shards" : {
        "total" : 2,
        "failed" : 0,
        "successful" : 2
    },
    "_index" : "twitter",
    "_type" : "tweet",
    "_id" : "1",
    "_version" : 1,
    "created" : true,
    "result" : created
}
```

_shards header提供有关索引操作的复制过程的信息。

- total - 表示应该执行索引操作的shard copies分片副本（主分片和副本分片）。
- success - 表示执行索引操作成功的shard copies分片副本（主分片和副本分片）。
- failed - 在副本分片上执行索引操作失败的情况下，包含与复制相关的错误的数组。

在索引操作成功的情况下，successful至少为1。

###### 注意

> 当索引操作成功返回时，副本分片可能并没有全部启动（默认情况下，仅主分片是必需的，但是此行为可以更改）。 在这种情况下，total将等于基于number_of_replicas设置的总分片数，successful将等于已启动的分片数（主分片和副本分片）。 如果没有失败，failed将为0。


### Automatic Index Creation（自动创建索引）

如果索引操作之前尚未创建，则自动创建索引（查看用于手动创建索引的create index API），如果还没有创建，还会自动为特定类型创建动态类型映射（请参阅 用于手动创建类型映射的put mapping API）。

映射本身非常灵活，是schema-free（无模式）的。 新的字段和对象将自动添加到指定类型的映射定义中。 有关映射定义的更多信息，请查看映射部分。

可以通过在所有节点的配置文件中将action.auto_create_index设置为false来禁用自动创建索引。 可以在每个索引设置通过将index.mapper.dynamic设置为false，可以禁用自动映射创建。

自动创建索引可以包括a pattern based white/black list（基于模式的白/黑名单列表），例如，将action.auto_create_index设置为+aaa*,-bbb*,+ccc*,-*（+表示允许，-表示不允许）。

### Versioning（版本控制）

每个索引文档都有一个版本号。 关联的version作为对index API请求的响应的一部分返回。 当指定version参数时，index API可以允许optimistic concurrency control（乐观的并发控制）。 这将控制要对其执行操作的文档的版本。 用于版本控制的用例的一个很好的例子是执行事务性的read-then-update（读取然后更新）。 指定version为从最初读取的文档中的version确保不会发生任何更改（为了更新而读取，建议将preference设置为_primary）。 例如：

```
PUT twitter/tweet/1?version=2
{
    "message" : "elasticsearch now has versioning support, double cool!"
}
```

注意：版本控制是完全实时的，不受近期实时搜索操作的影响。如果没有提供版本，则执行该操作，而不进行任何版本检查。

默认情况下，使用内部版本控制，从1开始，每个更新（包括删除）都会增加。可选地，可以使用外部值补充版本号（例如，如果在数据库中维护）。要启用此功能，version_type应设置为external。提供的值必须是数字，long类型，大于或等于0，小于大约9.2e+18。当使用外部版本类型时，系统将检查传递给索引请求的版本号是否大于当前存储的文档的版本号，而不是检查版本号是否匹配。如果为true，则文档将被索引并使用新的版本号。如果提供的值小于或等于存储的文档的版本号，则将发生版本冲突，并且索引操作将失败。

###### 警告

> 外部版本支持值0作为有效的版本号。 这允许版本与外部版本控制系统同步，其中版本号从0而不是1开始。 它具有副作用，版本号等于0的文档不能使用Update-By-Query API更新，也不能使用Delete By Query API删除，只要其版本号等于0。

一个很好的作用是，只要使用源数据库的版本号，就不需要对源数据库的更改结果执行异步索引操作的严格排序。 即使使用外部版本控制，也可以简化使用数据库中的数据更新Elasticsearch索引的简单情况，因为只有最新的版本号可以被使用，如果索引操作出于某种原因导致无序。

#### Version types（版本类型）

接下来说明上面的internal和external版本类型，Elasticsearch还支持特定用例的其他类型。 以下是不同版本类型及其语义的概述。

- internal

只有当给定版本号与存储文档的版本号相同时，才索引文档。

- external or external_gt

只有当给定版本号严格高于存储文档的版本号或者没有该文档不存在，才索引文档。 给定版本号将被用作新版本号，并将新版本号存储于新文档中。 提供的版本号必须是非负的long类型数字。

- external_gte

只有当给定版本号等于或高于存储文档的版本号，才索引文档。 如果该文档不存在，操作也将成功。 给定版本号将被用作新版本号，并将新版本号存储于新文档中。 提供的版本号必须是非负的long类型数字。

注意：external_gte版本类型适用于特殊用例，应谨慎使用。 如果使用不当，可能导致数据丢失。 还有一个选项是force，因为它会导致主分片和副本分片不一致，因此已被弃用。

### Operation Type（操作类型）

索引操作还接受op_type参数，可用于强制create操作，允许"put-if-absent"行为。 当使用create时，如果索引中已经存在该id的文档，则索引操作将失败。

以下是使用op_type参数的示例：

```
PUT twitter/tweet/1?op_type=create
{
    "user" : "kimchy",
    "post_date" : "2009-11-15T14:12:12",
    "message" : "trying out Elasticsearch"
}
```

指定create的另一个方式是使用以下uri：

```
PUT twitter/tweet/1/_create
{
    "user" : "kimchy",
    "post_date" : "2009-11-15T14:12:12",
    "message" : "trying out Elasticsearch"
}
```

### Automatic ID Generation（自动生成ID）

可以在不指定id的情况下执行索引操作。 在这种情况下，将自动生成ID。 此外，op_type将自动设置为create。 这是一个例子（注意使用POST而不是PUT）：

```
POST twitter/tweet/
{
    "user" : "kimchy",
    "post_date" : "2009-11-15T14:12:12",
    "message" : "trying out Elasticsearch"
}
```

上述索引操作的结果是：

```
{
    "_shards" : {
        "total" : 2,
        "failed" : 0,
        "successful" : 2
    },
    "_index" : "twitter",
    "_type" : "tweet",
    "_id" : "6a8ca01c-7896-48e9-81cc-9f70661fcb32",
    "_version" : 1,
    "created" : true,
    "result": "created"
}
```

### Routing（路由）

默认情况下，分片放置 - 或routing（路由） - 通过使用文档的id值的散列来控制。 为了更明确的控制，可以在每个操作的基础上使用routing参数直接指定路由器使用的散列函数的值。 例如：

```
POST twitter/tweet?routing=kimchy
{
    "user" : "kimchy",
    "post_date" : "2009-11-15T14:12:12",
    "message" : "trying out Elasticsearch"
}
```

在上面的示例中，"tweet"文档根据提供的routing参数路由到分片："kimchy"。

当设置显式映射时，可以选择使用_routing字段来指示索引操作从文档本身中提取路由值。 这会带来额外的文档解析传递的（非常小的）成本。 如果_routing映射被定义和设置为required，则如果没有提供或提取路由值，索引操作将失败。

### Parents & Children（父子关系）

可以通过在索引时指定其父文档来索引子文档。 例如：

```
PUT blogs
{
  "mappings": {
    "tag_parent": {},
    "blog_tag": {
      "_parent": {
        "type": "tag_parent"
      }
    }
  }
}

PUT blogs/blog_tag/1122?parent=1111
{
    "tag" : "something"
}
```

当索引子文档时，路由值将自动设置为与其父文档相同，除非使用routing参数明确指定路由值。

### Distributed（分布式）

索引操作基于其路由指向主分片（参见上面的Routing部分），并在包含此分片的实际节点上执行。 在主分片完成操作之后，如果需要，将更新分发到可用的副本。

### Wait For Active Shards（等待活动分片）

为了提高写入系统的弹性，索引操作可以配置为在继续操作之前等待一定数量的活动分片副本。如果所需数量的活动分片副本不可用，则写入操作必须等待并重试，直到必需的分片副本已启动或发生超时为止。默认情况下，写入操作只等待主分片处于活动状态，然后再继续（即：wait_for_active_shards = 1）。通过设置index.write.wait_for_active_shards可以动态地在索引设置中覆盖此默认值。要更改每个操作的这种行为，可以使用wait_for_active_shards请求参数。

有效值是all或任何正整数，（最多）达到索引中的每个分片的配置副本总数（即：number_of_replicas + 1）。指定一个负值或大于分片数量的数字将会引发错误。

例如，假设我们有一个集群，三个节点A，B和C，我们创建一个索引index，replicas副本数设置为3（导致4个分片副本，分片副本数比节点数多一个）。如果我们尝试索引操作，则默认情况下，操作将在确保每个分片的主分片可用后再继续。这意味着即使节点B和C挂了，节点A托管了主分片副本，索引操作仍可以继续进行，并且只有一个数据副本。如果在请求上设置了wait_for_active_shards到3（并且所有3个节点都已经启动），则索引操作将在进行之前需要3个活动分片副本，这是需要满足的要求，因为集群中有3个活动节点，每个节点都持有分片的副本。但是，如果我们将wait_for_active_shards设置为all（或到4，这是相同的），索引操作将不会继续进行，因为我们没有每个分片的所有4个副本在索引中处于活动状态。除非在集群中引入新节点以托管分片的第四个副本，否则操作将超时。

重要的是要注意，这种设置大大降低了写入操作没有写入必需数量的分片副本的机会，但是并不能完全消除这种可能性，因为这种检查是在写操作开始之前发生的。一旦写入操作正在进行中，仍然有可能复制在任何数量的分片副本上失败，但仍然在主分片成功。写入操作的响应的_shards部分显示复制成功/失败的分片副本的数量。

```
{
    "_shards" : {
        "total" : 2,
        "failed" : 0,
        "successful" : 2
    }
}
```

### Refresh（刷新）

控制此请求所做的更改对于搜索是可见的。 参阅Refresh。

### Noop Updates

当使用index API更新文档时，即使文档没有更改，也始终会为文档创建新版本号。 如果这是不可接受的，则将_update api的detect_noop参数设置为true。 这个选项在index api上不可用，因为index api没有获取旧的源，并且不能与新的源进行比较。

当noop updates是不可接受的时候，没有一个强制和快速的规则。 这是许多因素的组合，例如数据源发送实际上是noops的更新频率，以及Elasticsearch在分片上每秒运行的查询次数，接收更新的次数。

### Timeout（超时）

被分配执行索引操作的主分片在执行索引操作时可能不可用。 一些原因可能是主分片正在从网关恢复或正在进行迁移。 默认情况下，索引操作将等待主分片可用最多1分钟，然后才会出现错误并进行响应。 timeout参数可用于显式指定等待多长时间。 以下是将其设置为5分钟的示例：

```
PUT twitter/tweet/1?timeout=5m
{
    "user" : "kimchy",
    "post_date" : "2009-11-15T14:12:12",
    "message" : "trying out Elasticsearch"
}
```

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/5.4/docs-index_.html
