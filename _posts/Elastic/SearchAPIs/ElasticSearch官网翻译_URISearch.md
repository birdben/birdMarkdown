---
title: "ElasticSearch官网翻译_URISearch"
date: 2017-05-23 19:34:31
tags: [Elasticsearch]
categories: [Search]
---

#### URI Search

可以通过使用URI来提供请求参数执行搜索请求。 当使用此模式执行搜索时，并不是所有的搜索选项都会被公开，但它可以方便快速的"curl test"。 这是一个例子：

```
$ curl -XGET 'http://localhost:9200/twitter/tweet/_search?q=user:kimchy'
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

##### Parameters（参数）

URI中允许的参数有：

Name|Description
---|---
q|查询字符串（映射到query_string查询，有关详细信息，请参阅Query String查询）。
df|在查询中未定义字段前缀时使用的默认字段。
analyzer|分析查询字符串时使用的分析器名称。
lowercase_expanded_terms|应该将词条自动变成小写。 默认为true。
analyze_wildcard|应该分析通配符和前缀查询。 默认为false。
default_operator|要使用的默认运算符可以是AND或OR。 默认为OR。
lenient|如果设置为true，将会导致基于格式的失败（例如提供text类型到numeric字段）被忽略。 默认为false。
explain|对于每个命中，包含解释说明如何计算命中的评分。
_source|设置为false以禁用取回_source字段。 你还可以使用_source_include＆_source_exclude取回文档的一部分（有关详细信息，请参阅request body正文文档）
fields|要为每个命中返回的文档的选择store字段，以逗号分隔。 不指定任何值将不会返回任何字段。
sort|排序执行。 可以是fieldName，或者fieldName:asc/fieldName:desc的形式。 fieldName可以是文档中的实际字段，也可以是特殊_score名称，用于指示基于分数的排序。 可以有多个sort排序参数（顺序很重要）。
track_scores|当排序时，设置为true，以便仍然跟踪分数，并将它们作为每个命中的一部分返回。
timeout|搜索超时，限制在指定时间值内执行的搜索请求，并且保留与到期时累积的点击数。 默认为无超时。
terminate_after|每个分片收集的最大文档数量，到达此数量查询执行将提前终止。 如果设置，响应将有一个布尔字段terminate_early来指示查询执行是否实际已终止。 默认为无terminate_after。
from|从命中的索引开始返回。 默认为0。
size|要返回的命中数。 默认为10。
search_type|要执行的搜索操作的类型。 可以是dfs_query_then_fetch，query_then_fetch，scan [2.1.0弃用]，或count [2.0.0-beta1弃用]。 默认为query_then_fetch。 有关可执行的不同类型搜索的更多详细信息，请参阅搜索类型。

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/2.4/search-uri-request.html

