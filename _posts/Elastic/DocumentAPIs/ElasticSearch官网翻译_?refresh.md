---
title: "ElasticSearch官网翻译_?refresh"
date: 2017-06-01 19:16:29
tags: [Elasticsearch]
categories: [Search]
---

## ?refresh

Index，Update，Delete和Bulk API支持设置refresh以控制此请求所做的更改对搜索可见。这些是允许的值：

- Empty string or true

在操作发生后立即刷新相关的主分片和副本分片（而不是整个索引），以便更新的文档立即显示在搜索结果中。只有仔细思考和验证，从索引和搜索的角度出发，才不会导致性能不佳。

- wait_for

在响应返回之前等待请求所做的更改通过刷新对搜索可见。这不会强制立即刷新，而是等待刷新发生。 Elasticsearch会自动刷新已经更改的分片通过每个index.refresh_interval默认为1秒。该设置是动态的。调用Refresh API或将任何支持该API的refresh设置为true也将导致刷新，从而导致正在使用refresh=wait_for运行的请求返回。

- false (the default)

不设置刷新相关的动作。在请求返回后，此请求所做的更改将在某个时刻对搜索可见。

### Choosing which setting to use（选择使用哪种设置）

除非你有一个很好的理由等待更改变得可见，总是使用refresh=false，或者，因为这是默认值，只要不在URL中使用refresh参数。那是最简单和最快的选择。

如果你绝对必须使请求可见性与请求同步，那么你必须在对Elasticsearch进行更多的加载（true）和等待更长的响应（wait_for）之间进行选择。这里有几点应该告诉这个决定：

- 与true相比，wait_for能让索引做更多的变更工作。在这种情况下，每隔index.refresh_interval设置的时间，索引的修改才会保存一次。
- 设置为true将创建较少有效的索引构造（微小的段），以后必须将其合并到更有效的索引构造（较大的段）中。意思是说，设置为true的成本是在索引时间上支付，来创建微小的段，在搜索时搜索微小的段，并在合并时来制作较大的段。
- 不要在一行中启动多个refresh=wait_for请求。而是将它们批量写入到单个bulk请求，并且使用refresh=wait_for参数，并且Elasticsearch将并行启动它们，并且只有当它们全部完成时才返回。
- 如果refresh interval刷新间隔设置为-1，禁用自动刷新，则refresh=wait_for的请求将无限期地等待，直到某些操作导致刷新。相反，将index.refresh_interval设置为小于200ms的默认值会使refresh=wait_for更快地返回，但仍会生成低效的段。
- refresh=wait_for仅影响其所在的请求，但是，通过立即强制刷新，refresh=true将影响其他正在进行的请求。一般来说，如果你有一个运行的系统，你不想扰乱（其他请求），那么refresh=wait_for是一个较小的修改。

### refresh=wait_for Can Force a Refresh（refresh=wait_for可以导致强制刷新）

如果一个refresh=wait_for请求进来，当已经有index.max_refresh_listeners（默认为1000）请求等待该分片上的刷新时，那么该请求的行为就好像refresh设置为true：它将强制刷新。 这保证了当一个refresh=wait_for请求返回其更改对于搜索是可见的时候，同时防止阻塞请求对未检查的资源使用。 如果一个请求被强制刷新，因为它超出监听器slots，则其响应将包含"forced_refresh": true。

Bulk请求只占用每个分片上的一个slot，无论他们对分片修改多少次。

### Examples（例子）

这些将创建一个文档并立即刷新索引，使其可见：

```
PUT /test/test/1?refresh
{"test": "test"}
PUT /test/test/2?refresh=true
{"test": "test"}
```

这些将创建一个文档，而不做任何事情使其可以搜索：

```
PUT /test/test/3
{"test": "test"}
PUT /test/test/4?refresh=false
{"test": "test"}
```

这将创建一个文档并等待它变成对搜索可见：

```
PUT /test/test/4?refresh=wait_for
{"test": "test"}
```

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/5.4/docs-refresh.html
