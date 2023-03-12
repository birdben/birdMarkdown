---
title: "ElasticSearch官网翻译_ReindexAPI"
date: 2017-06-01 17:54:48
tags: [Elasticsearch]
categories: [Search]
---

## Reindex API

###### 重要

> Reindex不会尝试设置目标索引。 它不会复制源索引的设置。 你应该在运行_reindex操作之前设置目标索引，包括设置mappings映射，shard分片数量，replicas副本等。

_reindex的最基本形式只是将文档从一个索引复制到另一个索引。 这将会将文档从twitter索引复制到new_twitter索引中：

```
POST _reindex
{
  "source": {
    "index": "twitter"
  },
  "dest": {
    "index": "new_twitter"
  }
}
```

这样会返回：

```
{
  "took" : 147,
  "timed_out": false,
  "created": 120,
  "updated": 0,
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

就像_update_by_query一样，_reindex获取源索引的快照，但其目标必须是不同的索引，因此版本冲突是不可能的。 dest元素可以像index API一样进行配置，以控制optimistic concurrency control（乐观并发控制）。 只需将version_type（如上所述）或将其设置为internal将导致Elasticsearch盲目将文档转储到目标索引中，将覆盖具有相同类型和ID的任何内容：

```
POST _reindex
{
  "source": {
    "index": "twitter"
  },
  "dest": {
    "index": "new_twitter",
    "version_type": "internal"
  }
}
```

将version_type设置为external将导致Elasticsearch从源文档中保留版本，创建缺少的任何文档，并更新在目标索引中具有比源索引中更旧版本的任何文档：

```
POST _reindex
{
  "source": {
    "index": "twitter"
  },
  "dest": {
    "index": "new_twitter",
    "version_type": "external"
  }
}
```

设置op_type为create将导致_reindex仅在目标索引中创建丢失的文档。 所有现有文档将导致版本冲突：

```
POST _reindex
{
  "source": {
    "index": "twitter"
  },
  "dest": {
    "index": "new_twitter",
    "op_type": "create"
  }
}
```

默认情况下，版本冲突中止_reindex进程，但你可以通过在请求主体中设置"conflicts": "proceed"对版本冲突文档进行计数：

```
POST _reindex
{
  "conflicts": "proceed",
  "source": {
    "index": "twitter"
  },
  "dest": {
    "index": "new_twitter",
    "op_type": "create"
  }
}
```

你可以通过向source添加类型或添加查询来限制文档。 这只会将kimchy用户发布的tweet复制到new_twitter中：

```
POST _reindex
{
  "source": {
    "index": "twitter",
    "type": "tweet",
    "query": {
      "term": {
        "user": "kimchy"
      }
    }
  },
  "dest": {
    "index": "new_twitter"
  }
}
```

source中的index和type都可以是列表，允许你从一个请求中的大量来源复制。 这将从twitter和blog索引中的tweet和post类型中复制文档。 它会在twitter索引中包含post类型和blog索引中的tweet类型。 如果你想更具体，你将需要使用查询。 它也没有努力处理ID冲突。 目标索引将保持有效，但由于迭代顺序没有很好的定义，预测哪个文档可以存留是不容易的。

```
POST _reindex
{
  "source": {
    "index": ["twitter", "blog"],
    "type": ["tweet", "post"]
  },
  "dest": {
    "index": "all_together"
  }
}
```

还可以通过设置size来限制处理的文档的数量。 这只会将单个文档从twitter复制到new_twitter：

```
POST _reindex
{
  "size": 1,
  "source": {
    "index": "twitter"
  },
  "dest": {
    "index": "new_twitter"
  }
}
```

如果你想要一个特定的文档集从twitter索引你需要排序。 排序使scroll滚动效率更低，但在某些情况下它是值得的。 如果可能，更喜欢更多的选择性查询size和sort。 这将从twitter复制10000个文档到new_twitter：

```
POST _reindex
{
  "size": 10000,
  "source": {
    "index": "twitter",
    "sort": { "date": "desc" }
  },
  "dest": {
    "index": "new_twitter"
  }
}
```

source部分支持search request搜索请求中支持的所有元素。 例如，只有使用原始文档的一部分字段可以被reindex，通过使用源过滤如下所示：

```
POST _reindex
{
  "source": {
    "index": "twitter",
    "_source": ["user", "tweet"]
  },
  "dest": {
    "index": "new_twitter"
  }
}
```

像_update_by_query一样，_reindex支持修改文档的脚本。 与_update_by_query不同，脚本允许修改文档的元数据。 此示例颠覆了源文档的版本：

```
POST _reindex
{
  "source": {
    "index": "twitter"
  },
  "dest": {
    "index": "new_twitter",
    "version_type": "external"
  },
  "script": {
    "inline": "if (ctx._source.foo == 'bar') {ctx._version++; ctx._source.remove('foo')}",
    "lang": "painless"
  }
}
```

就像在_update_by_query中一样，你可以设置ctx.op来更改在目标索引上执行的操作：

- noop

如果你的脚本确定文档不必在目标索引中进行索引，请设置ctx.op = "noop"。 这个空操作将在响应主体的noop计数器字段中体现。

- delete

如果你的脚本决定必须删除该文档，请设置ctx.op = "delete"。删除将在响应主体的deleted计数器中体现。

将ctx.op设置为其他任何内容都是错误的。在ctx中设置任何其他字段也是错误的。

想想可能性！ 要小心点！ 很强大... 你可以改变：

- _id
- _type
- _index
- _version
- _routing
- _parent

将_version设置为null或从ctx map清除就像在index请求中不发送版本一样。 这将导致该目标索引中的文档被覆盖，无论目标版本或_reindex请求中使用的版本类型如何。

默认情况下，如果_reindex看到具有路由的文档，则路由将被保留，除非脚本被更改。 你可以根据dest请求设置routing路由来更改：

- keep

将每个匹配发送的批量请求的路由设置为匹配上的路由。 默认值。

- discard

将每个匹配发送的批量请求的路由设置为null。

- =<some text>

将每个匹配发送的批量请求的路由设置为=之后的所有文本。

例如，你可以使用以下请求将来自源索引的所有文档与公司名称cat复制到路由设置为cat的dest索引。

```
POST _reindex
{
  "source": {
    "index": "source",
    "query": {
      "match": {
        "company": "cat"
      }
    }
  },
  "dest": {
    "index": "dest",
    "routing": "=cat"
  }
}
```

默认情况下，_reindex的scroll批量大小为1000。你可以使用source元素中的size字段更改批量大小：

```
POST _reindex
{
  "source": {
    "index": "source",
    "size": 100
  },
  "dest": {
    "index": "dest",
    "routing": "=cat"
  }
}
```

Reindex也可以使用Ingest Node功能来指定如下pipeline管道：

```
POST _reindex
{
  "source": {
    "index": "source"
  },
  "dest": {
    "index": "dest",
    "pipeline": "some_ingest_pipeline"
  }
}
```

### Reindex from Remote（从远程索引Reindex）

###### 警告

> 从远程索引Reindex在5.4.0中被破坏，在5.4.1中修复。

Reindex支持从远程Elasticsearch群集重新索引：

```
POST _reindex
{
  "source": {
    "remote": {
      "host": "http://otherhost:9200",
      "username": "user",
      "password": "pass"
    },
    "index": "source",
    "query": {
      "match": {
        "test": "data"
      }
    }
  },
  "dest": {
    "index": "dest"
  }
}
```

host参数必须包含scheme（协议），host和port（例如https://otherhost:9200）。username和password参数是可选的，当它们存在时，重新索引将使用basic auth（基本认证）连接到远程Elasticsearch节点。使用基本认证时请务必使用https，密码将以纯文本格式发送。

必须使用reindex.remote.whitelist属性在elasticsearch.yaml中将远程主机明确列入白名单。它可以设置为允许的远程host和port组合的逗号分隔列表（例如otherhost:9200, another:9200, 127.0.10.*:9200, localhost:*）。白名单忽略了scheme（协议） - 仅使用主机和端口。

此功能应适用于你可能找到的任何版本的Elasticsearch的远程群集。这应该允许你从任何版本的Elasticsearch升级到当前版本，通过从旧版本的集群重新建立索引。

要启用发送到旧版本Elasticsearch的查询，query参数将直接发送到远程主机，无需验证或修改。

从远程服务器重新索引使用默认最大大小为100mb的堆栈缓冲区。如果远程索引包含非常大的文档，则需要使用较小的批量大小。下面的示例设置非常非常小的批量大小10。

```
POST _reindex
{
  "source": {
    "remote": {
      "host": "http://otherhost:9200"
    },
    "index": "source",
    "size": 10,
    "query": {
      "match": {
        "test": "data"
      }
    }
  },
  "dest": {
    "index": "dest"
  }
}
```

也可以使用socket_timeout字段在远程连接上设置socket读取超时，并使用connect_timeout字段设置连接超时。 两者默认为30秒。 此示例将套接字读取超时设置为1分钟，并将连接超时设置为10秒：

```
POST _reindex
{
  "source": {
    "remote": {
      "host": "http://otherhost:9200",
      "socket_timeout": "1m",
      "connect_timeout": "10s"
    },
    "index": "source",
    "query": {
      "match": {
        "test": "data"
      }
    }
  },
  "dest": {
    "index": "dest"
  }
}
```

### URL Parameters（URL参数）

除了像pretty这样的标准参数之外，Reindex API还支持refresh，wait_for_completion，wait_for_active_shards，timeout和requests_per_second。

发送refresh url参数将导致请求写入的所有索引都要被刷新。这与Index API的refresh参数不同，该参数只会导致接收到新数据的分片被刷新。

如果请求包含wait_for_completion=false，那么Elasticsearch将执行一些预检检查，启动请求，然后返回task，这个task可以使用Tasks API来取消或获取任务的状态。 Elasticsearch还将以.tasks/task/${taskId}作为文档创建此任务的记录。这是你可以根据是否合适来保留或删除它。当任务完成后删除它，这样Elasticsearch可以回收它使用的空间。

wait_for_active_shards控制在继续请求之前必须有多少分片副本处于活动状态。详见这里。timeout控制每个写入请求等待不可用分片变成可用状态的时间。两者都能正确地在Bulk API中工作。

requests_per_second可以设置为任何十进制正数（1.4,6,1000等），并且限制每秒执行一次reindex请求的次数，或者将其设置为-1以禁用限制。限流是在批次执行之间等待，以便它可以操纵scroll timeout。等待时间是批次完成的时间与requests_per_second * requests_in_the_batch的时间之间的差异。由于批次没有被分解成多个bulk request（批量请求），所以大的批次会导致Elasticsearch创建许多请求，然后在开始下一个集合之前等待一段时间。这是"bursty"而不是"smooth"。默认值为-1。

### Response body（响应主体）

JSON响应如下所示：

```
{
  "took" : 639,
  "updated": 0,
  "created": 123,
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

- created

已成功创建的文档数量。

- batches

reindex返回的scroll响应的数量。

- version_conflicts

reindex命中的版本冲突的数量。

- retries

reindex尝试的重试次数。 bulk是重试的批量操作的数量，search是重试的搜索操作的数量。

- throttled_millis

请求的毫秒数符合request_per_second的数量。

- failures

所有索引失败的数组。 如果这是非空的，那么请求因为这些失败而中止。 请参阅conflicts，如何防止版本冲突中止操作。

### Works with the Task API（使用Task API工作）

你可以使用Task API获取所有正在运行的reindex请求的状态：

```
GET _tasks?detailed=true&actions=*reindex
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
          "action" : "indices:data/write/reindex",
          "status" : {    
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
            },
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

任何Reindex都可以使用Task Cancel API取消：

```
POST _tasks/task_id:1/_cancel
```

可以使用上面的Task API找到task_id。

取消应尽快发生，但可能需要几秒钟。上面的Task Status API将继续列出任务，直到它被唤醒取消自身。

### Rethrottling（重新限流）

可以使用_rethrottle API在运行的reindex上更改requests_per_second的值：

```
POST _reindex/task_id:1/_rethrottle?requests_per_second=-1
```

可以使用上面的Task API找到task_id。

就像在_reindex API中设置它时一样，request_per_second可以是-1来禁用限流，或者任何十进制数字，如1.7或12，来限制到该级别。 加速查询的rethrottling会立即生效，但是减慢查询的rethrottling将在完成当前批处理之后生效。 这样可以防止scroll timeout滚动超时。

### Reindex to change the name of a field（重新索引更改字段名称）

_reindex可用于使用重命名的字段构建索引的副本。 假设你创建一个包含如下所示的文档的索引：

```
POST test/test/1?refresh
{
  "text": "words words",
  "flag": "foo"
}
```

但是你不喜欢flag这个名称，而是要用tag替换它。 _reindex可以为你创建其他索引：

```
POST _reindex
{
  "source": {
    "index": "test"
  },
  "dest": {
    "index": "test2"
  },
  "script": {
    "inline": "ctx._source.tag = ctx._source.remove(\"flag\")"
  }
}
```

现在你可以得到新的文件：

```
GET test2/test/1
```

它看起来像：

```
{
  "found": true,
  "_id": "1",
  "_index": "test2",
  "_type": "test",
  "_version": 1,
  "_source": {
    "text": "words words",
    "tag": "foo"
  }
}
```

或者你可以通过tag或任何你想要的搜索。

### Manual slicing（手动切片）

Reindex支持Sliced Scroll（切片滚动），你可以轻松地手动并行化进程：

```
POST _reindex
{
  "source": {
    "index": "twitter",
    "slice": {
      "id": 0,
      "max": 2
    }
  },
  "dest": {
    "index": "new_twitter"
  }
}
POST _reindex
{
  "source": {
    "index": "twitter",
    "slice": {
      "id": 1,
      "max": 2
    }
  },
  "dest": {
    "index": "new_twitter"
  }
}
```

你可以通过以下方式验证：

```
GET _refresh
POST new_twitter/_search?size=0&filter_path=hits.total
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

你还可以让reindex自动并行在_uid上使用Sliced Scroll（切片滚动）来进行切片：

```
POST _reindex?slices=5&refresh
{
  "source": {
    "index": "twitter"
  },
  "dest": {
    "index": "new_twitter"
  }
}
```

你还可以通过以下方式验证：

```
POST new_twitter/_search?size=0&filter_path=hits.total
```

结果是合理的total像这样：

```
{
  "hits": {
    "total": 120
  }
}
```

将slices添加到_reindex中可以自动执行上述部分中使用的手动过程，创建子请求，这意味着它有一些特殊的点：

- 你可以在Task API中看到这些请求。这些子请求是具有slices切片请求任务的子任务。
- 使用slices切片请求获取任务的状态只包含已完成切片的状态。
- 这些子请求可以单独寻址，例如取消和重新限流。
- 使用slices切片重新对请求限流将会按比例重新限制未完成的子请求。
- 使用slices切片取消请求将取消每个子请求。
- 由于slices切片的性质，每个子请求将不会获得完全均匀的文档部分。所有文档都将被处理，但有些切片可能比其他切片大。预期更大的切片具有更均匀的分布。
- 带有slices切片的请求的request_per_second和size的参数按比例分配给每个子请求。结合上述关于分布的不均匀性，你应该得出结论，使用slices切片size可能得到并不完全正确的size文档被"_reindex"ed。
- 每个子请求都会获得源索引的略有不同的快照，尽管这些都是大致相同的时间。

### Picking the number of slices（设置切片数）

在这一点上，我们围绕要使用的slices切片数量（如果手动并行化，则slice API中的max参数）提供了一些建议：

- 不要使用太大的数值。500就能造成CPU相当大的抖动。
- 从查询性能的角度来看，在源索引中使用分片数量的一些倍数更有效。
- 从查询性能的角度来看，在源索引中使用完全相同的分片是效率最高的。
- 索引性能应在可用资源之间以切片数量线性扩展。
- 索引或查询性能是否被该进程控制取决于许多因素，如正在重新索引的文档和集群正在进行重新索引。

### Reindex daily indices（重新索引每日索引）

你可以使用_reindex与Painless组合来重新索引每日索引，以将新模板应用于现有文档。

假设你有由以下文件组成的索引：

```
PUT metricbeat-2016.05.30/beat/1?refresh
{"system.cpu.idle.pct": 0.908}
PUT metricbeat-2016.05.31/beat/1?refresh
{"system.cpu.idle.pct": 0.105}
```

metricbeat-*索引的新模板已经加载到Elasticsearch中，但它仅适用于新创建的索引。 Painless可用于重新索引现有文档并应用新模板。

下面的脚本从索引名称中提取日期，并创建一个（原索引名称）后面添加-1的新索引。 来自metricbeat-2016.05.31的所有数据将重新索引到metricbeat-2016.05.31-1。

```
POST _reindex
{
  "source": {
    "index": "metricbeat-*"
  },
  "dest": {
    "index": "metricbeat"
  },
  "script": {
    "lang": "painless",
    "inline": "ctx._index = 'metricbeat-' + (ctx._index.substring('metricbeat-'.length(), ctx._index.length())) + '-1'"
  }
}
```

前面的metricbeat索引的所有文档现在可以通过*-1在索引中找到。

```
GET metricbeat-2016.05.30-1/beat/1
GET metricbeat-2016.05.31-1/beat/1
```

前面的方法也可以与change the name of a field（更改字段的名称）一起使用，以便将现有数据加载到新索引中，但如果需要，还可以重命名字段。

### Extracting a random subset of an index（提取索引的随机子集）

Reindex可用于提取的索引的随机子集用于测试：

```
POST _reindex
{
  "size": 10,
  "source": {
    "index": "twitter",
    "query": {
      "function_score" : {
        "query" : { "match_all": {} },
        "random_score" : {}
      }
    },
    "sort": "_score"    (1)
  },
  "dest": {
    "index": "random_twitter"
  }
}
```

- (1) Reindex默认按_doc排序，所以random_score不会有任何效果，除非你将排序重写为_score。

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/5.4/docs-reindex.html
