---
title: "ElasticSearch官网翻译_GetAPI"
date: 2017-05-31 17:22:50
tags: [Elasticsearch]
categories: [Search]
---

## Get API

Get API允许根据其id从索引中获取一个类型的JSON文档。 以下示例从名为twitter的索引获取一个名为tweet的类型的JSON文档，其ID为0：

```
GET twitter/tweet/0
```

上述操作的结果是：

```
{
    "_index" : "twitter",
    "_type" : "tweet",
    "_id" : "0",
    "_version" : 1,
    "found": true,
    "_source" : {
        "user" : "kimchy",
        "date" : "2009-11-15T14:12:12",
        "likes": 0,
        "message" : "trying out Elasticsearch"
    }
}
```

上述结果包括我们希望取回的文档的_index，_type，_id和_version，包括如果可以找到文档的实际_source（如响应中found的字段所示）。

API还允许使用HEAD检查文档的存在，例如：

```
HEAD twitter/tweet/0
```

### Realtime（实时的）

默认情况下，Get API是实时的，不受索引刷新率的影响（数据将在搜索时可见）。 如果文档已更新但尚未刷新，则Get API将发出刷新调用in-place以使文档可见。 这也将使上一次刷新后更改的其他文档可见。 为了禁用实时GET，可以将realtime参数设置为false。

### Optional Type（可选的类型）

Get API允许_type是可选的。 将其设置为_all，以便获取与所有类型的id匹配的第一个文档。

### Source filtering（源过滤器）

默认情况下，Get操作返回_source字段的内容，除非你使用stored_fields参数或者_source字段被禁用。 你可以使用_source参数关闭_source检索：

```
GET twitter/tweet/0?_source=false
```

如果你只需要完整_source中的一个或两个字段，你可以使用_source_include和_source_exclude参数来包含或过滤出你需要的部分。 这对于大型文档尤其有用，部分取回可以节省网络开销。 这两个参数取逗号分隔的字段或通配符表达式列表。 例：

```
GET twitter/tweet/0?_source_include=*.id&_source_exclude=entities
```

如果只想指定include，可以使用较短的表示法：

```
GET twitter/tweet/0?_source=*.id,retweeted
```

### Stored Fields（存储字段）

Get操作允许指定将通过传递stored_fields参数返回的一组存储字段。 如果请求的字段未被存储，它们将被忽略。 例如考虑以下映射：

```
PUT twitter
{
   "mappings": {
      "tweet": {
         "properties": {
            "counter": {
               "type": "integer",
               "store": false
            },
            "tags": {
               "type": "keyword",
               "store": true
            }
         }
      }
   }
}
```

现在我们可以添加一个文档：

```
PUT twitter/tweet/1
{
    "counter" : 1,
    "tags" : ["red"]
}
```

1. 并尝试取回它：

```
GET twitter/tweet/1?stored_fields=tags,counter
```

上述操作的结果是：

```
{
   "_index": "twitter",
   "_type": "tweet",
   "_id": "1",
   "_version": 1,
   "found": true,
   "fields": {
      "tags": [
         "red"
      ]
   }
}
```

从文档中获取的字段值自身始终作为数组返回。 由于counter字段未被存储，所以Get请求在尝试获取stored_field时简单地忽略它。

也可以取回_routing和_parent字段之类的元数据字段：

```
PUT twitter/tweet/2?routing=user1
{
    "counter" : 1,
    "tags" : ["white"]
}
```

```
GET twitter/tweet/2?routing=user1&stored_fields=tags,counter
```

上述操作的结果是：

```
{
   "_index": "twitter",
   "_type": "tweet",
   "_id": "2",
   "_version": 1,
   "_routing": "user1",
   "found": true,
   "fields": {
      "tags": [
         "white"
      ]
   }
}
```

只可以通过stored_field选项返回叶子字段。 因此，对象字段无法返回，并且这样的请求将失败。

### Generated fields（生成的字段）

如果在索引和刷新之间没有刷新，GET将访问transaction log（事务日志）以获取文档。 但是，有些字段只有当索引时才会生成。 如果你尝试访问这些仅在索引时生成的字段，则会得到异常（默认）。 如果通过设置ignore_errors_on_generated_fields=true来访问transaction log（事务日志），则可以选择忽略生成的字段。

### Getting the _source directly（直接获取_source）

使用/{index}/{type}/{id}/_source endpoint来获取文档的_source字段，而没有任何附加内容。 例如：

```
GET twitter/tweet/1/_source
```

你还可以使用相同的源过滤器参数来控制_source的哪些部分将被返回：

```
GET twitter/tweet/1/_source?_source_include=*.id&_source_exclude=entities'
```

请注意，_source endpoint还有一个HEAD变体来高效地测试文档_source的存在。 如果在映射中禁用了现有文档，则它将不具有_source。

```
HEAD twitter/tweet/1/_source
```

### Routing（路由）

当使用控制routing路由的能力进行索引时，为了获取文档，还应提供routing路由值。 例如：

```
GET twitter/tweet/2?routing=user1
```

以上将收到带有id为2的tweet，但将根据用户进行routing路由。 注意，如果发出的获取请求没有正确的routing路由，将导致文档不能被获取。

### Preference（首选项）

控制哪些分片副本执行Get请求的preference。 默认情况下，操作在分片副本之间是随机执行的。

preference可以设置为：

- _primary

该操作将仅在主分片上执行。

- _local

如果可能，操作将优选在本地分配的分片上执行。

- Custom (string) value

将使用自定义值来确保相同的分片将用于相同的自定义值。 这可以帮助在不同刷新状态下命中不同分片时的"jumping values"。 示例值可以与Web Session ID或用户名类似。

### Refresh（刷新）

refresh参数可以设置为true，以便在Get操作之前刷新相关的分片，并使其可搜索。 将其设置为true应仔细思考和验证后，（以确定）不会对系统造成重负荷（并减慢索引）。

### Distributed（分布式）

Get操作被哈希化为特定的分片ID。 然后将其重定向到该分片ID中的一个副本，并返回结果。 副本是该分片ID组中的主分片及其副本分片。 这意味着我们将拥有更多的副本，我们将拥有更好的GET扩展。

### Versioning support（版本控制支持）

只有当当前版本号等于指定版本号时，才能使用version参数来取回文档。 对于所有版本类型而言，此行为是相同的，但版本类型是FORCE的除外，它始终取回文档。 请注意，FORCE版本类型已弃用。

在内部，Elasticsearch已将旧文档标记为已删除，并添加了一个全新的文档。 文档的旧版本不会立即消失，尽管你将无法访问该文档。 当你继续索引更多数据时，Elasticsearch会在后台清理已删除的文档。

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/5.4/docs-get.html
