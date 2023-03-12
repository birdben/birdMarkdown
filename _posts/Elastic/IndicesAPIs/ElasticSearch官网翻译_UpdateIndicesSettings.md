---
title: "ElasticSearch官网翻译_UpdateIndicesSettings"
date: 2017-06-05 10:53:06
tags: [Elasticsearch]
categories: [Search]
---

## Update Indices Settings

实时更改特定的索引级别设置。

REST endpoint是/_settings（更新所有索引）或者{index}/_settings以更新一个（或多个）索引设置。 请求的正文包括更新的设置，例如：

```
PUT /twitter/_settings
{
    "index" : {
        "number_of_replicas" : 2
    }
}
```

可以在（Index Modules）索引模块中找到可以在存活索引上动态更新的每个索引设置的列表。 要保留现有设置不被更新，可以将preserve_existing请求参数设置为true。

### Bulk Indexing Usage（批量索引用法）

例如，update settings API可用于动态地更改索引，对于批量索引而言更具性能，然后将其移动到更实时的索引状态。 批量索引开始之前，请使用：

```
PUT /twitter/_settings
{
    "index" : {
        "refresh_interval" : "-1"
    }
}
```

（另一个优化选项是启动没有任何副本的索引，只有稍后添加它们，但这真的取决于用例）。

然后，一旦批量索引完成，可以更新设置（设置回默认值）：

```
PUT /twitter/_settings
{
    "index" : {
        "refresh_interval" : "1s"
    }
}
```

而且，强制合并应该被调用：

```
POST /twitter/_forcemerge?max_num_segments=5
```

### Updating Index Analysis（更新索引分析器）

也可以为索引定义新的分析器。 但是，首先关闭索引，并在进行更改后将其打开。

例如，如果content分析器尚未在myindex上定义，但您可以使用以下命令来添加它：

```
POST /twitter/_close

PUT /twitter/_settings
{
  "analysis" : {
    "analyzer":{
      "content":{
        "type":"custom",
        "tokenizer":"whitespace"
      }
    }
  }
}

POST /twitter/_open
```

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/5.4/indices-update-settings.html
