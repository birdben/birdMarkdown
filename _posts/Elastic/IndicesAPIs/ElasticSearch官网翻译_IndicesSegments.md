---
title: "ElasticSearch官网翻译_IndicesSegments"
date: 2017-06-05 16:26:56
tags: [Elasticsearch]
categories: [Search]
---

## Indices Segments

提供Lucene索引（分片级）构建的低级别段信息。 允许用于提供关于分片和索引的状态的更多信息，可能的优化信息，删除引起的数据“浪费”等。

Endpoints包括特定索引，几个索引或全部的段：

```
curl -XGET 'http://localhost:9200/test/_segments'
curl -XGET 'http://localhost:9200/test1,test2/_segments'
curl -XGET 'http://localhost:9200/_segments'
```

响应：

```
{
    ...
        "_3": {
            "generation": 3,
            "num_docs": 1121,
            "deleted_docs": 53,
            "size_in_bytes": 228288,
            "memory_in_bytes": 3211,
            "committed": true,
            "search": true,
            "version": "4.6",
            "compound": true
        }
    ...
}
```

- _0

JSON文档的key是段的名称。 此名称用于生成文件名称：分片目录中的以此段名开始的所有文件属于此段。

- generation

当需要写入新的段时，generation的数值会递增。段名称源于这个generation数值。

- num_docs

存储在此段中的未删除文档的数量。

- deleted_docs

存储在此段中的已删除文档的数量。 如果这个数字大于0，那么当这个段被合并时，空间将被回收，这是非常好的。

- size_in_bytes

该段使用的磁盘空间量（以字节为单位）。

- memory_in_bytes

该段需要将一些数据存储到内存中以便搜索更加高效。 此数字返回用于此目的的字节数。 值为-1表示Elasticsearch无法计算此数字。

- committed

该段是否已在磁盘上同步。已经提交的段可以在硬重启后保留。 没有必要担心是false情况，未提交的段的数据也存储在transaction log事务日志中，以便Elasticsearch能够在下一次启动时reply（重新执行更改）。

- search

该段是否可搜索。 false值很可能意味着该段已经写入磁盘，但是之后没有执行刷新让它可以被搜索。

- version

已经用于写入此段的Lucene版本。

- compound

该段是否存储在复合文件中。 当为true时，这意味着Lucene将段中的所有文件合并到一个文件中，以保存文件描述符。

### Verbose mode（调试模式）

要添加可用于调试的其他信息，请使用verbose标志。

###### 警告

> 附加详细信息的格式是试验阶段的，可随时更改

```
curl -XGET 'http://localhost:9200/test/_segments?verbose=true'
```

响应：

```
{
    ...
        "_3": {
            ...
            "ram_tree": [
                {
                    "description": "postings [PerFieldPostings(format=1)]",
                    "size_in_bytes": 2696,
                    "children": [
                        {
                            "description": "format 'Lucene50_0' ...",
                            "size_in_bytes": 2608,
                            "children" :[ ... ]
                        },
                        ...
                    ]
                },
                ...
                ]

        }
    ...
}
```

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/5.4/indices-segments.html
