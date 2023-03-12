---
title: "ElasticSearch官网翻译_MultiSearchAPI"
date: 2017-05-28 11:21:41
tags: [Elasticsearch]
categories: [Search]
---

#### Multi Search API

Multi search API允许在同一API内执行多个搜索请求。 它的endpoint是_msearch。

请求的格式类似于bulk API格式，结构如下（如果特定搜索结束重定向到另一个节点，则结构被特别优化以减少解析）：

```
header\n
body\n
header\n
body\n
```

标题部分包括要搜索的index/indices，可选的（映射）类型，search_type，preference和routing。 body包括典型的搜索体请求（包括query，aggregations，from，size等）。 这是一个例子：

```
$ cat requests
{"index" : "test"}
{"query" : {"match_all" : {}}, "from" : 0, "size" : 10}
{"index" : "test", "search_type" : "dfs_query_then_fetch"}
{"query" : {"match_all" : {}}}
{}
{"query" : {"match_all" : {}}}

{"query" : {"match_all" : {}}}
{"search_type" : "dfs_query_then_fetch"}
{"query" : {"match_all" : {}}}

$ curl -XGET localhost:9200/_msearch --data-binary "@requests"; echo
```

注意，上面包括一个空header的示例（也可以只是没有任何内容）也被支持。

响应返回responses数组，其中包括与原始多重搜索请求中的顺序相匹配的每个搜索请求的搜索响应。 如果该特定搜索请求完全失败，将返回具有错误消息的对象，而不是实际的搜索响应。

该endpoint还可以针对URI本身的index/indices和type/types进行搜索，在这种情况下，它将被用作默认值，除非在header中另有明确定义。 例如：

```
$ cat requests
{}
{"query" : {"match_all" : {}}, "from" : 0, "size" : 10}
{}
{"query" : {"match_all" : {}}}
{"index" : "test2"}
{"query" : {"match_all" : {}}}

$ curl -XGET localhost:9200/test/_msearch --data-binary @requests; echo
```

上述将针对没有定义索引的所有请求对test索引执行搜索，最后一个将针对test2索引执行。

可以以类似的方式设置search_type，以全局应用于所有搜索请求。

##### Security

参阅URL-based access control


参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/2.4/search-multi-search.html

