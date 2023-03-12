---
title: "ElasticSearch官网翻译_UpdateByQueryAPI"
date: 2017-06-01 11:21:46
tags: [Elasticsearch]
categories: [Search]
---

## Update By Query API

_update_by_query的最简单的用法只是对索引中的每个文档执行更新，而不会更改源。 这对于picking up new properties（接收新的属性）或其他在线映射更改非常有用。 这是API：

```
POST twitter/_update_by_query?conflicts=proceed
```

会返回像这样：

```
{
  "took" : 147,
  "timed_out": false,
  "updated": 120,
  "deleted": 0,
  "batches": 1,
  "version_conflicts": 0,
  "noops": 0,
  "retries": {
    "bulk": 0,
    "search": 0
  },
  "throttled_millis": 0,
  "requests_per_second": -1.0,
  "throttled_until_millis": 0,
  "total": 120,
  "failures" : [ ]
}
```

_update_by_query在启动时获取索引的快照，并使用internal内部版本控制对其进行索引。 这意味着如果文档在进行快照和处理索引请求之间发生变化，则会发生版本冲突。 当版本匹配文档被更新并且版本号增加。

###### 注意

> 由于内部版本控制不支持0作为有效的版本号，版本等于0的文档无法使用_update_by_query进行更新，并且会使请求失败。

所有更新和查询失败导致_update_by_query中止并在响应中的failures返回错误信息。已执行的更新仍然坚持。换句话说，执行没有回滚，只会中止。当第一个故障导致中止时，批量请求失败返回的所有错误信息都会在failures元素中;因此，有可能会有不少失败的实体。

如果你想简单地计算版本冲突的数量，而不是导致它们中止，那么在URL中设置conflicts=proceed，或者在请求体中设置"conflicts": "proceed"。第一个例子是这样做，因为它只是尝试获取一个当前在线映射的变化，而版本冲突只是意味着这个冲突的文档已经被更新，在_update_by_query开始与尝试更新文档的时间之间。这很好，因为该更新将获取在线映射的更新。

回到API格式，你可以将_update_by_query限制为单一类型。这只会从Twitter的索引更新tweet文档：

```
POST twitter/tweet/_update_by_query?conflicts=proceed
```

你还可以使用Query DSL限制_update_by_query。 这将更新用户kimchy的twitter索引中的所有文档：

```
POST twitter/_update_by_query?conflicts=proceed
{
  "query": { (1)
    "term": {
      "user": "kimchy"
    }
  }
}
```

- (1) 该查询必须与Search API相同的方式，以query key的值传递。 你也可以以与search api相同的方式使用q参数。

到目前为止，我们只是更新文档而不改变它们的来源。 这对于picking up new properties（接收新的属性）来说真的很有用，但这只是一半的乐趣。 _update_by_query支持script脚本对象来更新文档。 这将增加所有kimchy的tweets上的likes字段的值：

```
POST twitter/_update_by_query
{
  "script": {
    "inline": "ctx._source.likes++",
    "lang": "painless"
  },
  "query": {
    "term": {
      "user": "kimchy"
    }
  }
}
```

就像Update API一样，你可以设置ctx.op来更改执行的操作：

- noop

如果你的脚本决定不必进行任何更改，请设置ctx.op = "noop"。这将导致_update_by_query从其更新中忽略该文档。这个空操作将在响应主体的noop计数器字段中体现。

- delete

如果你的脚本决定必须删除该文档，请设置ctx.op = "delete"。删除将在响应主体的deleted计数器中体现。

将ctx.op设置为其他任何内容都是错误的。在ctx中设置任何其他字段也是错误的。

请注意，我们停止指定conflict = proceed。在这种情况下，我们希望版本冲突中止该进程，以便我们可以处理该故障。

该API不允许你移动其修改的文档，只需修改它们的源。这是故意的！我们没有规定将文档从原始位置删除。

也可以一次性对多个索引和多个类型进行整体处理，就像search API一样：

```
POST twitter,blog/tweet,post/_update_by_query
```

如果你提供routing路由，则将routing路由复制到scroll query，将进程限制为与该路由值匹配的分片：

```
POST twitter/_update_by_query?routing=1
```

默认情况下_update_by_query使用scroll批量大小为1000。你可以使用URL参数scroll_size更改批量大小：

```
POST twitter/_update_by_query?scroll_size=100
```

_update_by_query也可以使用Ingest Node功能来指定pipeline管道，如下所示：

```
PUT _ingest/pipeline/set-foo
{
  "description" : "sets foo",
  "processors" : [ {
      "set" : {
        "field": "foo",
        "value": "bar"
      }
  } ]
}
POST twitter/_update_by_query?pipeline=set-foo
```

### URL Parameters（URL参数）

除了像pretty这样的标准参数之外，Update By Query API还支持refresh，wait_for_completion，wait_for_active_shards和timeout。

一旦请求完成，发送refresh参数将刷新所有被更新索引的分片。这不同于Index API的refresh参数，只是接收到新数据的分片被索引。

如果请求包含wait_for_completion=false，那么Elasticsearch将执行一些预检检查，启动请求，然后返回一个task，这个task可以使用Tasks API来取消或获取任务的状态。 Elasticsearch还将以.tasks/task/${taskId}作为文档创建此任务的记录。这是你可以根据是否合适来保留或删除它。当任务完成后删除它，这样Elasticsearch可以回收它使用的空间。

wait_for_active_shards控制在继续请求之前必须有多少分片副本处于活动状态。详见这里。timeout控制每个写入请求等待不可用分片变成可用状态的时间。两者都能正确地在Bulk API中工作。

requests_per_second可以设置为任何十进制正数（1.4,6,1000等），并且限制每秒执行一次update-by-query请求的次数，或者将其设置为-1以禁用限制。限流是在批次执行之间等待，以便它可以操纵scroll timeout。等待时间是批次完成的时间与requests_per_second * requests_in_the_batch的时间之间的差异。由于批次没有被分解成多个bulk request（批量请求），所以大的批次会导致Elasticsearch创建许多请求，然后在开始下一个集合之前等待一段时间。这是"bursty"而不是"smooth"。默认值为-1。

### Response body

JSON响应如下所示：

```
{
  "took" : 639,
  "updated": 0,
  "batches": 1,
  "version_conflicts": 2,
  "retries": {
    "bulk": 0,
    "search": 0
  }
  "throttled_millis": 0,
  "failures" : [ ]
}
```

- took

从整个操作的开始到结束的毫秒数。

- updated

已成功更新的文档数量。

- batches

update by query返回的scroll响应的数量。

- version_conflicts

update by query命中的版本冲突的数量。

- retries

update by query尝试的重试次数。 bulk是重试的批量操作的数量，search是重试的搜索操作的数量。

- throttled_millis

请求的毫秒数符合request_per_second的数量。

- failures

所有索引失败的数组。 如果这是非空的，那么请求因为这些失败而中止。 请参阅conflicts，如何防止版本冲突中止操作。

### Works with the Task API（使用Task API工作）

你可以使用Task API获取所有正在运行的update-by-query的状态：

```
GET _tasks?detailed=true&actions=*byquery
```

响应如下：

```
{
  "nodes" : {
    "r1A2WoRbTwKZ516z6NEs5A" : {
      "name" : "r1A2WoR",
      "transport_address" : "127.0.0.1:9300",
      "host" : "127.0.0.1",
      "ip" : "127.0.0.1:9300",
      "attributes" : {
        "testattr" : "test",
        "portsfile" : "true"
      },
      "tasks" : {
        "r1A2WoRbTwKZ516z6NEs5A:36619" : {
          "node" : "r1A2WoRbTwKZ516z6NEs5A",
          "id" : 36619,
          "type" : "transport",
          "action" : "indices:data/write/update/byquery",
          "status" : {    (1)
            "total" : 6154,
            "updated" : 3500,
            "created" : 0,
            "deleted" : 0,
            "batches" : 4,
            "version_conflicts" : 0,
            "noops" : 0,
            "retries": {
              "bulk": 0,
              "search": 0
            }
            "throttled_millis": 0
          },
          "description" : ""
        }
      }
    }
  }
}
```

- (1) 此对象包含实际状态。 它就像是响应json，附加重要的total字段。 total是reindex希望执行的操作总数。 你可以通过附带的updated，created和deleted的字段来估计进度。当它们的总和等于total字段时，请求将完成。

使用任务id可以直接查找任务：

```
GET /_tasks/taskId:1
```

这个API的优点是它与wait_for_completion=false集成，以透明地返回已完成任务的状态。 如果任务完成并且wait_for_completion=false，那么它将返回results或error字段。 此功能的成本是wait_for_completion=false在.tasks/task/${taskId}创建的文档。由你自己删除该文件。

### Works with the Cancel Task API（使用Cancel Task API工作）

任何Update By Query都可以使用Task Cancel API取消：

```
POST _tasks/task_id:1/_cancel
```

可以使用上面的Task API找到task_id。

取消应尽快发生，但可能需要几秒钟。上面的Task Status API将继续列出任务，直到它被唤醒取消自身。

### Rethrottling（重新限流）

request_per_second的值可以使用_rethrottle API更改正在运行的update by query：

```
POST _update_by_query/task_id:1/_rethrottle?requests_per_second=-1
```

可以使用上面的Task API找到task_id。

就像在_update_by_query API中设置它时一样，request_per_second可以是-1来禁用限流，或者任何十进制数字，如1.7或12，来限制到该级别。 加速查询的rethrottling会立即生效，但是减慢查询的rethrottling将在完成当前批处理之后生效。 这样可以防止scroll timeout滚动超时。

### Manual slicing（手动切片）

Update-by-query支持Sliced Scroll（切片滚动），你可以轻松地手动并行化进程：

```
POST twitter/_update_by_query
{
  "slice": {
    "id": 0,
    "max": 2
  },
  "script": {
    "inline": "ctx._source['extra'] = 'test'"
  }
}
POST twitter/_update_by_query
{
  "slice": {
    "id": 1,
    "max": 2
  },
  "script": {
    "inline": "ctx._source['extra'] = 'test'"
  }
}
```

你可以通过以下方式验证：

```
GET _refresh
POST twitter/_search?size=0&q=extra:test&filter_path=hits.total
```

结果是合理的total像这样：

```
{
  "hits": {
    "total": 120
  }
}
```

### Automatic slicing（自动切片）

你还可以让update-by-query自动并行在_uid上使用Sliced Scroll（切片滚动）来进行切片：

```
POST twitter/_update_by_query?refresh&slices=5
{
  "script": {
    "inline": "ctx._source['extra'] = 'test'"
  }
}
```

你还可以通过以下方式验证：

```
POST twitter/_search?size=0&q=extra:test&filter_path=hits.total
```

结果是合理的total像这样：

```
{
  "hits": {
    "total": 120
  }
}
```

将slices添加到_update_by_query中可以自动执行上述部分中使用的手动过程，创建子请求，这意味着它有一些特殊的点：

- 你可以在Task API中看到这些请求。这些子请求是具有slices切片请求任务的子任务。
- 使用slices切片请求获取任务的状态只包含已完成切片的状态。
- 这些子请求可以单独寻址，例如取消和重新限流。
- 使用slices切片重新对请求限流将会按比例重新限制未完成的子请求。
- 使用slices切片取消请求将取消每个子请求。
- 由于slices切片的性质，每个子请求将不会获得完全均匀的文档部分。所有文档都将被处理，但有些切片可能比其他切片大。预期更大的切片具有更均匀的分布。
- 带有slices切片的请求的request_per_second和size的参数按比例分配给每个子请求。结合上述关于分布的不均匀性，你应该得出结论，使用slices切片size可能得到并不完全正确的size文档被"_update_by_query"ed。
- 每个子请求都会获得源索引的略有不同的快照，尽管这些都是大致相同的时间。

### Picking the number of slices（设置切片数）

在这一点上，我们围绕要使用的slices切片数量（如果手动并行化，则slice API中的max参数）提供了一些建议：

- 不要使用太大的数值。500就能造成CPU相当大的抖动。
- 从查询性能的角度来看，在源索引中使用分片数量的一些倍数更有效。
- 从查询性能的角度来看，在源索引中使用完全相同的分片是效率最高的。
- 索引性能应在可用资源之间以切片数量线性扩展。
- 索引或查询性能是否被该进程控制取决于许多因素，如正在重新索引的文档和集群正在进行重新索引。

### Pick up a new property（接收新的属性）

假设你创建了一个没有dynamic mapping动态映射的索引，用数据填充它，然后添加一个映射值来从数据中获取更多的字段：

```
PUT test
{
  "mappings": {
    "test": {
      "dynamic": false,   (1)
      "properties": {
        "text": {"type": "text"}
      }
    }
  }
}

POST test/test?refresh
{
  "text": "words words",
  "flag": "bar"
}
POST test/test?refresh
{
  "text": "words words",
  "flag": "foo"
}
PUT test/_mapping/test   (2)
{
  "properties": {
    "text": {"type": "text"},
    "flag": {"type": "text", "analyzer": "keyword"}
  }
}
```

- (1) 这意味着新的字段将不会被索引，只存储在_source中。
- (2) 这将更新映射以添加新的flag字段。 要接收新的字段，你必须重新索引所有的文档。

搜索数据将找不到任何内容：

```
POST test/_search?filter_path=hits.total
{
  "query": {
    "match": {
      "flag": "foo"
    }
  }
}
```

```
{
  "hits" : {
    "total" : 0
  }
}
```

但是你可以发出_update_by_query请求来接收新映射：

```
POST test/_update_by_query?refresh&conflicts=proceed
POST test/_search?filter_path=hits.total
{
  "query": {
    "match": {
      "flag": "foo"
    }
  }
}
```

```
{
  "hits" : {
    "total" : 1
  }
}
```

当添加一个字段到multifield多字段时，你可以执行完全相同的操作。

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/5.4/docs-update-by-query.html
