---
title: "ElasticSearch官网翻译_OpenCloseIndexAPI"
date: 2017-06-03 15:17:23
tags: [Elasticsearch]
categories: [Search]
---

## Open / Close Index API

open/close index API可以关闭索引，稍后打开它。 关闭的索引在集群上几乎没有开销（除了维护其元数据），并且对于索引的读/写操作都被阻止。 可以打开一个关闭的索引，然后通过正常的恢复过程。

REST endpoint是/{index}/_close和/{index}/_open。 例如：

```
POST /my_index/_close

POST /my_index/_open
```

可以打开和关闭多个索引。 如果请求明确引用不存在的索引，则会抛出错误。 可以使用ignore_unavailable=true参数禁用此行为。

可以使用_all作为索引名称或指定全部（例如*）标识它们的模式，可以一次打开或关闭所有索引。

通过将配置文件中的action.destructive_requires_name标志设置为true，可以禁用通过通配符或_all标识索引。 也可以通过集群update settings api来更改此设置。

关闭的索引将消耗大量的磁盘空间，这可能会导致托管环境中的问题。 将cluster.indices.close.enable设置为false可以通过集群设置API禁用关闭索引。 默认值为true。

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/5.4/indices-open-close.html
