---
title: "ElasticSearch官网翻译_NamedQueries"
date: 2017-05-26 01:32:11
tags: [Elasticsearch]
categories: [Search]
---

#### Named Queries

每个filter过滤器和query查询都可以在其顶层定义中接受_name参数。

```
{
    "bool" : {
        "should" : [
            {"match" : { "name.first" : {"query" : "shay", "_name" : "first"} }},
            {"match" : { "name.last" : {"query" : "banon", "_name" : "last"} }}
        ],
        "filter" : {
            "terms" : {
                "name.last" : ["banon", "kimchy"],
                "_name" : "test"
            }
        }
    }
}
```

搜索响应将包括每个命中匹配的match_queries。 查询和过滤器的标记只对bool query有意义。

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/2.4/search-request-named-queries-and-filters.html

