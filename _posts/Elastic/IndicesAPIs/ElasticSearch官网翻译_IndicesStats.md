---
title: "ElasticSearch官网翻译_IndicesStats"
date: 2017-06-05 17:02:43
tags: [Elasticsearch]
categories: [Search]
---

## Indices Stats

Indices level stats（索引级别统计）信息提供了索引上发生的不同操作的统计信息。 API提供了索引级别范围的统计信息（尽管大多数统计信息也可以使用节点级范围检索）。

以下内容返回所有索引的high level aggregation（高级聚合）和index level（索引级）统计信息：

```
GET /_stats
```

可以使用以下方式检索特定索引统计信息

```
GET /index1,index2/_stats
```

默认情况下，返回所有统计信息，只有当URI中特殊指定返回的统计信息。 这些数据可以是以下任何一种：

值|描述
---|---
docs|文档/已删除文档的数量（尚未合并的文档）。 注意，受刷新索引影响。
store|索引的大小。
indexing|Index索引统计信息可以与逗号分隔的types类型列表组合，以提供文档类型级统计信息。
get|Get获取统计信息，包括missing统计信息。
search|Search搜索统计资料，包括suggest建议统计资料。 你可以通过添加额外的groups组参数（搜索操作可以与一个或多个组关联）来包括自定义组的统计信息。 groups参数接受逗号分隔的组名列表。 使用_all返回所有组的统计信息。
segments|获取开放段的内存使用情况。 或者，设置include_segment_file_sizes标志报告Lucene索引文件中每一个的聚合磁盘使用情况。
completion|完成suggest建议统计信息。
fielddata|Fielddata统计信息。
flush|Flush统计信息。
merge|Merge统计信息。
request_cache|分片请求缓存统计信息。
refresh|Refresh统计信息。
warmer|Warmer统计信息。
translog|Translog统计信息。

一些统计信息允许每个字段的粒度，其接受包括字段的逗号分隔列表的列表。 默认情况下，所有字段都包括在内：

值|描述
---|---
fields|要列入统计信息的字段列表。 除非提供更具体的字段列表，否则将其用作默认列表（见下文）。
completion_fields|要包括在Completion Suggest statistics（完成建议统计信息）中的字段列表。
fielddata_fields|要包括在Fielddata统计信息中的字段列表。

以下是一些示例：

```
# Get back stats for merge and refresh only for all indices
GET /_stats/merge,refresh
# Get back stats for type1 and type2 documents for the my_index index
GET /my_index/_stats/indexing?types=type1,type2
# Get back just search stats for group1 and group2
GET /_stats/search?groups=group1,group2
```

返回的统计信息在索引级别聚合，其中包含primaries和total聚合，其中，primaries中的值只是主分片的值，total是主分片和副本分片的累积值。

为了获取分片级别的统计信息，请将level参数设置为shards。

注意，随着分片在群集周围移动，它们的统计信息将在其他节点上创建时被清除。 另一方面，即使分片"left"一个节点，该节点仍然保留分片贡献的统计数据。

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/5.4/indices-stats.html
