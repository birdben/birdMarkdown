---
title: "ElasticSearch官网翻译_UpdateAPI"
date: 2017-05-31 22:16:21
tags: [Elasticsearch]
categories: [Search]
---

## Update API

Update API允许根据提供的脚本来更新文档。 操作从索引获取文档（与分片搭配），运行脚本（具有可选脚本语言和参数），并将结果建立索引（也允许删除或忽略该操作）。 它使用版本控制来确保在"get"和"reindex"期间没有发生更新。

请注意，此操作仍然意味着文档的完整重新索引，它只会减少一些网络往返，并减少get和index之间版本冲突的机会。 需要启用_source字段以使此功能正常工作。

例如，让我们索引一个简单的文档：

```
PUT test/type1/1
{
    "counter" : 1,
    "tags" : ["red"]
}
```

### Scripted updates（脚本更新）

现在，我们可以执行一个增加计数器的脚本：

```
POST test/type1/1/_update
{
    "script" : {
        "inline": "ctx._source.counter += params.count",
        "lang": "painless",
        "params" : {
            "count" : 4
        }
    }
}
```

我们可以在tags列表中添加一个tag（注意，如果tag存在，它将会添加它，因为它是列表）：

```
POST test/type1/1/_update
{
    "script" : {
        "inline": "ctx._source.tags.add(params.tag)",
        "lang": "painless",
        "params" : {
            "tag" : "blue"
        }
    }
}
```

除_source外，以下变量可通过ctx映射获得：_index，_type，_id，_version，_routing，_parent和_now（当前时间戳）。

我们还可以在文档中添加一个新的字段：

```
POST test/type1/1/_update
{
    "script" : "ctx._source.new_field = \"value_of_new_field\""
}
```

或从文档中删除一个字段：

```
POST test/type1/1/_update
{
    "script" : "ctx._source.remove(\"new_field\")"
}
```

而且，我们甚至可以改变执行的操作。 此示例删除文档，如果tags字段包含green，否则它什么都不做（noop）：

```
POST test/type1/1/_update
{
    "script" : {
        "inline": "if (ctx._source.tags.contains(params.tag)) { ctx.op = \"delete\" } else { ctx.op = \"none\" }",
        "lang": "painless",
        "params" : {
            "tag" : "green"
        }
    }
}
```

### Updates with a partial document（更新部分文档）

Update API还支持传递一个部分文档，它将被合并到现有文档中（简单的递归合并，对象的内部合并，替换核心"keys/values"和数组）。 例如：

```
POST test/type1/1/_update
{
    "doc" : {
        "name" : "new_name"
    }
}
```

如果指定了doc和scripts都指定了，则忽略doc。 最好是将部分文档的字段对放在脚本本身中。

### Detecting noop updates（检测无改变更新）

如果指定了doc，它的值将与现有的_source合并。 默认情况下，不更改任何内容的更新会检测到它们不会更改任何内容并返回"result":"noop"，如下所示：

```
POST test/type1/1/_update
{
    "doc" : {
        "name" : "new_name"
    }
}
```

如果在发送请求之前name是new_name，则忽略整个更新请求。 如果请求被忽略，则响应中的result元素返回noop。

```
{
   "_shards": {
        "total": 0,
        "successful": 0,
        "failed": 0
   },
   "_index": "test",
   "_type": "type1",
   "_id": "1",
   "_version": 6,
   "result": noop
}
```

你可以通过设置"detect_noop":false来禁用此行为如下所示：

```
POST test/type1/1/_update
{
    "doc" : {
        "name" : "new_name"
    },
    "detect_noop": false
}
```

### Upserts（更新插入）

如果文档不存在，则将upsert的内容作为新的文档插入。 如果文档确实存在，那么script将被执行：

```
POST test/type1/1/_update
{
    "script" : {
        "inline": "ctx._source.counter += params.count",
        "lang": "painless",
        "params" : {
            "count" : 4
        }
    },
    "upsert" : {
        "counter" : 1
    }
}
```

#### scripted_upsert

如果你希望脚本运行，无论文档是否存在，即脚本处理初始化文档而不是upsert元素，则将scripted_upsert设置为true：

```
POST sessions/session/dh3sgudg8gsrgl/_update
{
    "scripted_upsert":true,
    "script" : {
        "id": "my_web_session_summariser",
        "params" : {
            "pageViewEvent" : {
                "url":"foo.com/bar",
                "response":404,
                "time":"2014-01-01 12:32"
            }
        }
    },
    "upsert" : {}
}
```

#### doc_as_upsert

将doc_as_upsert设置为true，将使用doc的内容用作upsert值，而不是发送部分doc加上upsert文档：

```
POST test/type1/1/_update
{
    "doc" : {
        "name" : "new_name"
    },
    "doc_as_upsert" : true
}
```

### Parameters

更新操作支持以下查询字符串参数：

值|描述
---|---
retry_on_conflict|在更新的获取和索引阶段之间，另一个进程可能已经更新了同一个文档。 默认情况下，更新将失败并出现版本冲突异常。 retry_on_conflict参数控制在最后抛出异常之前重试更新的次数。
routing|如果正在更新的文档不存在，则使用路由将更新请求路由到正确的分片，并为upsert请求设置路由。 不能用于更新现有文档的路由。
parent|如果正在更新的文档不存在，则parent用于将更新请求路由到正确的分片，并为upsert请求设置parent。 不能用于更新现有文档的parent。 如果指定了别名索引路由，则它将覆盖parent路由，并用于路由请求。
timeout|超时等待分片变为可用。
wait_for_active_shards|在进行更新操作之前，需要处于活动状态的分片副本的数量。 详见这里。
refresh|控制此请求所做的更改对于搜索是可见的。 参阅?refresh。
_source|允许控制在响应中如何返回更新的源。 默认情况下，不会返回更新的源。 有关详细信息，请参阅Source Filtering。
version & version_type|Update API在内部使用Elasticsearch的版本控制支持，以确保在更新过程中文档不会更改。 你可以使用version参数指定仅当文档的版本与指定的版本匹配时才应更新该文档。 通过将version type版本类型设置为force，你可以在更新后强制使用新版本的文档（谨慎使用！force不保证文档没有更改）。

###### 注意：The update API does not support external versioning（更新API不支持外部版本控制）

> Update API不支持外部版本控制（版本类型external＆external_gte），因为它会导致Elasticsearch版本号与外部系统不同步。 使用index API代替。

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/5.4/docs-update.html
