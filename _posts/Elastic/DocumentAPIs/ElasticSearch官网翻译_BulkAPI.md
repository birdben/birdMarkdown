---
title: "ElasticSearch官网翻译_BulkAPI"
date: 2017-06-01 17:46:47
tags: [Elasticsearch]
categories: [Search]
---

## Bulk API

Bulk API可以在单个API调用中执行许多index/delete操作。 这可以大大提高索引速度。

###### Client support for bulk requests（客户端支持批量请求）

> 一些官方支持的客户端提供工具来协助从一个索引到另一个索引的bulk请求和重新索引：
> 
> - Perl
>
> 请参阅Search::Elasticsearch::Bulk和Search::Elasticsearch::Scroll
>
> - Python
> 
> 参见elasticsearch.helpers.*

REST API endpoint是/_bulk，它期望以下换行符分隔JSON（NDJSON）结构：

```
action_and_meta_data\n
optional_source\n
action_and_meta_data\n
optional_source\n
....
action_and_meta_data\n
optional_source\n
```

注意：行尾数据必须以换行符\n结尾。 每个换行符字符之前都可以回车\r。 当向该endpoint发送请求时，应将Content-Type头设置为application/x-ndjson。

可能的操作是index，create，delete和update。 index和create需要下一行是源，并且具有与标准index API的op_type参数相同的语义（即，如果具有相同索引和类型的文档已经存在，则create将失败，而index将根据需要添加或替换文档）。 delete并不需要下一行是源，并且具有与标准delete API相同的语义。 update需要在下一行指定部分文档，upsert和script及其选项。

如果要提供文本文件作为curl命令的输入，则必须使用--data-binary标志，而不是plain -d。 后者不保留换行符。 例：

```
$ cat requests
{ "index" : { "_index" : "test", "_type" : "type1", "_id" : "1" } }
{ "field1" : "value1" }
$ curl -s -H "Content-Type: application/x-ndjson" -XPOST localhost:9200/_bulk --data-binary "@requests"; echo
{"took":7, "errors": false, "items":[{"index":{"_index":"test","_type":"type1","_id":"1","_version":1,"result":"created","forced_refresh":false}}]}
```

因为此格式使用文字\n作为分隔符，请确保JSON操作和源不会pretty打印。 以下是批量命令正确序列的示例：

```
POST _bulk
{ "index" : { "_index" : "test", "_type" : "type1", "_id" : "1" } }
{ "field1" : "value1" }
{ "delete" : { "_index" : "test", "_type" : "type1", "_id" : "2" } }
{ "create" : { "_index" : "test", "_type" : "type1", "_id" : "3" } }
{ "field1" : "value3" }
{ "update" : {"_id" : "1", "_type" : "type1", "_index" : "test"} }
{ "doc" : {"field2" : "value2"} }
```

此批量操作的结果是：

```
{
   "took": 30,
   "errors": false,
   "items": [
      {
         "index": {
            "_index": "test",
            "_type": "type1",
            "_id": "1",
            "_version": 1,
            "result": "created",
            "_shards": {
               "total": 2,
               "successful": 1,
               "failed": 0
            },
            "created": true,
            "status": 201
         }
      },
      {
         "delete": {
            "found": false,
            "_index": "test",
            "_type": "type1",
            "_id": "2",
            "_version": 1,
            "result": "not_found",
            "_shards": {
               "total": 2,
               "successful": 1,
               "failed": 0
            },
            "status": 404
         }
      },
      {
         "create": {
            "_index": "test",
            "_type": "type1",
            "_id": "3",
            "_version": 1,
            "result": "created",
            "_shards": {
               "total": 2,
               "successful": 1,
               "failed": 0
            },
            "created": true,
            "status": 201
         }
      },
      {
         "update": {
            "_index": "test",
            "_type": "type1",
            "_id": "1",
            "_version": 2,
            "result": "updated",
            "_shards": {
                "total": 2,
                "successful": 1,
                "failed": 0
            },
            "status": 200
         }
      }
   ]
}
```

endpoints是/_bulk，/{index}/_bulk和{index}/{type}/_bulk。当提供index或index/type时，默认情况下它们将不会明确地提供给它们的批量条目使用。

关于格式的注意。这里的想法是尽可能快地处理这个问题。由于某些操作将被重定向到其他节点上的其他分片，因此在接收节点侧仅解析action_meta_data。

使用此协议的客户端库应尽可能尝试在客户端执行类似操作，并尽可能减少缓冲。

对批量操作的响应是一个大的JSON结构，其中显示了每个操作的各个结果。一个动作的失败不会影响剩余的动作。

在一个单独的bulk call中没有“正确”的执行条目数量。你应该尝试使用不同的设置来找到特定工作负载的最佳大小。

如果使用HTTP API，请确保客户端不发送HTTP块，因为这会使速度减慢。

### Versioning（版本控制）

每个批量项目可以使用_version/version字段包含版本值。 它基于_version映射自动跟踪index/delete操作的行为。 它还支持version_type/_version_type（请参阅versioning）

### Routing（路由）

每个批量项目可以使用_routing/routing字段包括路由值。 它基于_routing映射自动跟踪index/delete操作的行为。

### Parent（父级）

每个批量项目可以使用_parent/parent字段来包含父级。 它基于_parent/_routing映射自动跟踪index/delete操作的行为。

### Wait For Active Shards（等待活动分片）

进行批量调用时，您可以设置wait_for_active_shards参数，以便在开始处理批量请求之前要求最小数量的分片副本处于活动状态。 有关详细信息和使用示例，请参阅此处。

### Refresh（刷新）

控制此请求所做的更改对于搜索是可见的。 参见refresh。

### Update（更新）

当使用update操作_retry_on_conflict可以用作动作本身的字段（而不是额外的有效payload行）时，可以指定在版本冲突的情况下应重试更新的次数。

Update操作有效payload支持以下选项：doc（部分文档），upsert，doc_as_upsert，script，params（脚本使用），lang（脚本使用）和_source。 有关选项的详细信息，请参阅更新文档。 更新操作的示例：

```
POST _bulk
{ "update" : {"_id" : "1", "_type" : "type1", "_index" : "index1", "_retry_on_conflict" : 3} }
{ "doc" : {"field" : "value"} }
{ "update" : { "_id" : "0", "_type" : "type1", "_index" : "index1", "_retry_on_conflict" : 3} }
{ "script" : { "inline": "ctx._source.counter += params.param1", "lang" : "painless", "params" : {"param1" : 1}}, "upsert" : {"counter" : 1}}
{ "update" : {"_id" : "2", "_type" : "type1", "_index" : "index1", "_retry_on_conflict" : 3} }
{ "doc" : {"field" : "value"}, "doc_as_upsert" : true }
{ "update" : {"_id" : "3", "_type" : "type1", "_index" : "index1", "_source" : true} }
{ "doc" : {"field" : "value"} }
{ "update" : {"_id" : "4", "_type" : "type1", "_index" : "index1"} }
{ "doc" : {"field" : "value"}, "_source": true}
```

### Security

见 URL-based access control

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/5.4/docs-bulk.html
