---
title: "ElasticSearch官网翻译_DeleteAPI"
date: 2017-05-31 15:15:50
tags: [Elasticsearch]
categories: [Search]
---

## Delete By Query API

_delete_by_query的最简单的用法只是在与查询匹配的每个文档上执行删除。 这是API：

```
POST twitter/_delete_by_query
{
  "query": { (1)
    "match": {
      "message": "some message"
    }
  }
}
```

- (1) 该查询必须与Search API相同的方式，以query key的值传递。 你也可以以与search api相同的方式使用q参数。

这样会响应：

```
{
  "took" : 147,
  "timed_out": false,
  "deleted": 119,
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
  "total": 119,
  "failures" : [ ]
}
```

_delete_by_query在启动时获取索引的快照，并使用internal内部版本控制删除它所发现的内容。这意味着如果文档在进行快照和处理删除请求之间发生变化，将发生版本冲突。当版本匹配时文档会被删除。

###### 注意

> 由于internal内部版本控制不支持0作为有效的版本号，因此无法使用_delete_by_query删除版本号等于0的文档，并且将请求失败。

在_delete_by_query执行期间，依次执行多个搜索请求，以便找到要删除的所有匹配文档。每次发现一批文档时，执行相应的批量请求以删除所有这些文档。如果搜索或批量请求被拒绝，_delete_by_query依赖于默认策略来重试拒绝的请求（最多10次，以指数返回）。达到最大重试次数限制会导致_delete_by_query中止，并在响应的failures中返回所有错误信息。已经执行的删除仍然坚持。换句话说，执行没有回滚，只会中止。当第一个故障导致中止时，批量请求失败返回的所有错误信息都会在failures元素中被返回;因此，有可能会有不少失败的实体。

如果你想计算版本冲突的数量，而不是导致它们中止，那么在URL中设置conflicts=proceed，或者在请求体中设置"conflicts": "proceed"。

回到API格式，你可以将_delete_by_query限制为单一类型。这只会从Twitter的索引中删除tweet类型文档：

```
POST twitter/tweet/_delete_by_query?conflicts=proceed
{
  "query": {
    "match_all": {}
  }
}
```

也可以一次删除多个索引和多个类型的文档，就像search API一样：

```
POST twitter,blog/tweet,post/_delete_by_query
{
  "query": {
    "match_all": {}
  }
}
```

如果你提供routing路由，则将routing路由复制到scroll查询，将进程限制为与该路由值匹配的分片：

```
POST twitter/_delete_by_query?routing=1
{
  "query": {
    "range" : {
        "age" : {
           "gte" : 10
        }
    }
  }
}
```

默认情况下_delete_by_query使用scroll批量大小为1000。你可以使用URL参数scroll_size更改批量大小：

```
POST twitter/_delete_by_query?scroll_size=5000
{
  "query": {
    "term": {
      "user": "kimchy"
    }
  }
}
```

### URL Parameters（URL参数）

除了像pretty这样的标准参数之外，Delete By Query API还支持refresh，wait_for_completion，wait_for_active_shards和timeout。

一旦请求完成，发送refresh参数将刷新所有涉及delete by query操作的分片。这与Delete API的refresh参数不同，只是接收到删除请求的分片会被刷新。

如果请求包含wait_for_completion = false，那么Elasticsearch将执行一些预检检查，启动请求，然后返回一个task，这个task可以使用Tasks API来取消或获取任务的状态。 Elasticsearch还将以.tasks/task/${taskId}作为文档创建此任务的记录。这是你可以根据是否合适来保留或删除它。当任务完成后删除它，这样Elasticsearch可以回收它使用的空间。

wait_for_active_shards控制在继续请求之前必须有多少分片副本处于活动状态。详见这里。timeout控制每个写入请求等待不可用分片变成可用状态的时间。两者都能正确地在Bulk API中工作。

requests_per_second可以设置为任何十进制正数（1.4,6,1000等），并且限制每秒执行一次delete-by-query请求的次数，或者将其设置为-1以禁用限制。限流是在批次执行之间等待，以便它可以操纵scroll timeout。等待时间是批次完成的时间与requests_per_second * requests_in_the_batch的时间之间的差异。由于批次没有被分解成多个bulk request（批量请求），所以大的批次会导致Elasticsearch创建许多请求，然后在开始下一个集合之前等待一段时间。这是"bursty"而不是"smooth"。默认值为-1。

### Response body（响应主体）

JSON响应如下所示：

```
{
  "took" : 639,
  "deleted": 0,
  "batches": 1,
  "version_conflicts": 2,
  "retries": 0,
  "throttled_millis": 0,
  "failures" : [ ]
}
```

- took

从整个操作的开始到结束的毫秒数。

- deleted

已成功删除的文档数量。

- batches

delete by query返回的scroll响应的数量。

- version_conflicts

delete by query命中的版本冲突的数量。

- retries

delete by query的重试次数是响应于完整队列。

- throttled_millis

请求的毫秒数符合request_per_second的数量。

- failures

所有索引失败的数组。 如果这是非空的，那么请求因为这些失败而中止。 请参阅conflicts，如何防止版本冲突中止操作。

### Works with the Task API（使用Task API工作）

你可以使用Task API获取任何正在运行的delete-by-query的状态：

```
GET _tasks?detailed=true&actions=*/delete/byquery
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
          "action" : "indices:data/write/delete/byquery",
          "status" : {    (1)
            "total" : 6154,
            "updated" : 0,
            "created" : 0,
            "deleted" : 3500,
            "batches" : 36,
            "version_conflicts" : 0,
            "noops" : 0,
            "retries": 0,
            "throttled_millis": 0
          },
          "description" : ""
        }
      }
    }
  }
}
```

- (1) 此对象包含实际状态。它就像是响应json，附加重要的total字段。 total是reindex希望执行的操作总数。你可以通过附带的updated，created和deleted的字段来估计进度。当它们的总和等于total字段时，请求将完成。

使用任务id可以直接查找任务：

```
GET /_tasks/taskId:1
```

这个API的优点是它与wait_for_completion=false集成，以透明地返回已完成任务的状态。如果任务完成并且wait_for_completion=false被设置，那么它将返回results或error字段。此功能的成本是wait_for_completion=false在.tasks/task/${taskId}创建的文档。由你自己删除该文件。

### Works with the Cancel Task API（使用Cancel Task API工作）

任何Delete By Query可以使用Task Cancel API取消：

```
POST _tasks/task_id:1/_cancel
```

可以使用上面的Task API找到task_id。

取消应尽快发生，但可能需要几秒钟。上面的Task Status API将继续列出任务，直到它被唤醒取消自身。

### Rethrottling（重新限流）

request_per_second的值可以使用_rethrottle API更改正在运行的delete by query：

```
POST _delete_by_query/task_id:1/_rethrottle?requests_per_second=-1
```

可以使用上面的Task API找到task_id。

就像在_delete_by_query API中设置它一样，request_per_second可以是-1来禁用限流，或者任何十进制数字，如1.7或12，来限制到该级别。 加速查询的rethrottling会立即生效，但是减慢查询的rethrottling将在完成当前批处理之后生效。 这样可以防止scroll timeout滚动超时。

### Manually slicing（手动切片）

Delete-by-query支持Sliced Scroll（切片滚动），你可以轻松地手动并行化进程：

```
POST twitter/_delete_by_query
{
  "slice": {
    "id": 0,
    "max": 2
  },
  "query": {
    "range": {
      "likes": {
        "lt": 10
      }
    }
  }
}
POST twitter/_delete_by_query
{
  "slice": {
    "id": 1,
    "max": 2
  },
  "query": {
    "range": {
      "likes": {
        "lt": 10
      }
    }
  }
}
```

你可以通过以下方式验证：

```
GET _refresh
POST twitter/_search?size=0&filter_path=hits.total
{
  "query": {
    "range": {
      "likes": {
        "lt": 10
      }
    }
  }
}
```

结果是合理的total像这样：

```
{
  "hits": {
    "total": 0
  }
}
```

### Automatic slicing（自动切片）

你还可以让delete-by-query自动并行在_uid上使用Sliced Scroll（切片滚动）来进行切片：

```
POST twitter/_delete_by_query?refresh&slices=5
{
  "query": {
    "range": {
      "likes": {
        "lt": 10
      }
    }
  }
}
```

你还可以通过以下方式验证：

```
POST twitter/_search?size=0&filter_path=hits.total
{
  "query": {
    "range": {
      "likes": {
        "lt": 10
      }
    }
  }
}
```

结果是合理的total像这样：

```
{
  "hits": {
    "total": 0
  }
}
```

将slices添加到_delete_by_query中可以自动执行上述部分中使用的手动过程，创建子请求，这意味着它有一些特殊的点：

- 你可以在Task API中看到这些请求。这些子请求是具有slices切片请求任务的子任务。
- 使用slices切片请求获取任务的状态只包含已完成切片的状态。
- 这些子请求可以单独寻址，例如取消和重新限流。
- 使用slices切片重新对请求限流将会按比例重新限制未完成的子请求。
- 使用slices切片取消请求将取消每个子请求。
- 由于slices切片的性质，每个子请求将不会获得完全均匀的文档部分。所有文档都将被处理，但有些切片可能比其他切片大。预期更大的切片具有更均匀的分布。
- 带有slices切片的请求的request_per_second和size的参数按比例分配给每个子请求。结合上述关于分布的不均匀性，你应该得出结论，使用slices切片size可能得到并不完全正确的size文档被"_delete_by_query"ed。
- 每个子请求都会获得源索引的略有不同的快照，尽管这些都是大致相同的时间。

### Picking the number of slices（设置切片数）

在这一点上，我们围绕要使用的slices切片数量（如果手动并行化，则slice API中的max参数）提供了一些建议：

- 不要使用太大的数值。500就能造成CPU相当大的抖动。
- 从查询性能的角度来看，在源索引中使用分片数量的一些倍数更有效。
- 从查询性能的角度来看，在源索引中使用完全相同的分片是效率最高的。
- 索引性能应在可用资源之间以切片数量线性扩展。
- 索引或查询性能是否被该进程控制取决于许多因素，如正在重新索引的文档和集群正在进行重新索引。

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/5.4/docs-delete-by-query.html
