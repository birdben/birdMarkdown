---
title: "ElasticSearch官网翻译_RequestBodySearch"
date: 2017-05-24 11:13:34
tags: [Elasticsearch]
categories: [Search]
---

#### Request Body Search（请求主体搜索）

搜索请求可以被执行使用搜索DSL，包括查询DSL在请求体内。 这是一个例子：

```
$ curl -XGET 'http://localhost:9200/twitter/tweet/_search' -d '{
    "query" : {
        "term" : { "user" : "kimchy" }
    }
}'
```

这里是一个示例响应：

```
{
    "_shards":{
        "total" : 5,
        "successful" : 5,
        "failed" : 0
    },
    "hits":{
        "total" : 1,
        "hits" : [
            {
                "_index" : "twitter",
                "_type" : "tweet",
                "_id" : "1",
                "_source" : {
                    "user" : "kimchy",
                    "postDate" : "2009-11-15T14:12:12",
                    "message" : "trying out Elasticsearch"
                }
            }
        ]
    }
}
```

##### 参数

参数|描述
---|---
timeout|搜索超时，限制在指定时间值内执行的搜索请求，并且保留与到期时累积的点击数。 默认为无超时。 请参阅“Time units”一节。
from|从某个偏移中返回命中结果。 默认为0。
size|要返回的命中数。 默认为10。 如果你不关心获取一些命中结果，但仅关心匹配和/或聚合的数量，将值设置为0将有助于提高性能。
search_type|要执行的搜索操作的类型。 可以是dfs_query_then_fetch或query_then_fetch。 默认为query_then_fetch。 查看Search Type了解更多。
request_cache|设置为true或false以启用或禁用size为0的请求的搜索结果的缓存，即aggregations聚合和suggestions建议（不返回顶部命中）。 请参阅Shard request cache。
terminate_after|每个分片收集的最大文档数量，到达此数量查询执行将提前终止。 如果设置，响应将有一个boolean字段terminate_early来指示查询执行是否实际已终止。 默认为terminate_after。

在上述中，search_type和request_cache必须作为query-string查询字符串参数传递。 搜索请求的其余部分应在主体本身内传递。 主体内容也可以作为名为source的REST参数传递。

HTTP GET和HTTP POST都可以用来执行与body的搜索。 由于并非所有客户端都支持GET，所以POST也是允许的。

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/2.4/search-request-body.html

