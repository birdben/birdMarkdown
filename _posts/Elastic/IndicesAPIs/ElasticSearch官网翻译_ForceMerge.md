---
title: "ElasticSearch官网翻译_ForceMerge"
date: 2017-06-06 01:26:29
tags: [Elasticsearch]
categories: [Search]
---

## Force Merge

Force merge API允许通过API强制合并一个或多个索引。 合并涉及Lucene索引在每个分片中保存的段数。 强制合并操作允许通过合并来减少段数。

此调用将阻止，直到合并完成。 如果http连接丢失，则请求将在后台继续，并且任何新请求将被阻塞，直到之前的强制合并完成。

```
POST /twitter/_forcemerge
```

### Request Parameters（请求参数）

force merge API接受以下请求参数：

值|描述
---|---
max_num_segments|合并之后的段数量。 要完全合并索引，请将其设置为1。默认为只需检查合并是否需要执行，如果是，则执行它。
only_expunge_deletes|合并过程只能在其中删除段删除。 在Lucene中，文档不会从段中删除，只被标记为已删除。 在段的合并过程中，创建没有已被标记为删除的新段。 此标志仅允许合并具有删除的段。 默认为false。 请注意，这不会覆盖index.merge.policy.expunge_deletes_allowed阈值。
flush|是否在强制合并后执行flush。默认为true。

### Multi Index（多索引）

强制合并API可以通过单个调用应用于多个索引，甚至可以应用于_all索引。

```
POST /kimchy,elasticsearch/_forcemerge

POST /_forcemerge
```

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/5.4/indices-forcemerge.html
