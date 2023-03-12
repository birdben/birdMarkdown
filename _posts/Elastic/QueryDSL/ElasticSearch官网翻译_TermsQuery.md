---
title: "ElasticSearch官网翻译_TermQuery"
date: 2017-06-23 21:21:02
tags: [Elasticsearch]
categories: [Search]
---

## Terms Query

过滤文档具有与任何提供的词条（未分析）匹配的字段。 例如：

```
GET /_search
{
    "query": {
        "constant_score" : {
            "filter" : {
                "terms" : { "user" : ["kimchy", "elasticsearch"]}
            }
        }
    }
}
```

Terms Query也使用in作为过滤器别名，以便更简单的使用[5.0.0中弃用，使用terms]。

##### Terms lookup mechanism（terms查找机制）

当需要使用大量词条来指定terms过滤器时，从索引中的文档中获取这些词条值可能是有益的。 一个具体的例子是过滤你的follower的推特的推文。 潜在的在terms过滤器中指定的用户ID的数量可能会很多。 在这种情况下，使用terms过滤器的terms lookup mechanism（terms查找机制）是有意义的。

terms lookup mechanism（terms查找机制）支持以下选项：

值|描述
---|---
index|从索引中获取词条的值。 默认为当前索引。
type|从类型中获取词条值的值。
id|从文档的ID中获取词条的值。
path|通过指定路径的字段来获取terms过滤器的实际值。
routing|检索外部词条文档时使用的自定义路由值。

terms过滤器的值将从指定索引的指定类型的指定ID的文档中的字段中获取。 在内部执行获取请求以从指定的路径获取值。 在此功能工作的时刻，_source需要被存储。

此外，如果“引用”词条数据不大，请考虑使用带有单个分片的索引，并在所有节点上完全复制。 查找terms过滤器将更愿意在本地节点上执行获取请求，如果可能，减少了对网络的需要。

##### Terms lookup twitter example（terms查询twitter示例）

首先，我们为id为2的用户索引信息，特别是其followers，然后从id为1的用户索引一个tweet。最后，我们搜索与用户2的followers匹配的所有tweets。

```
PUT /users/user/2
{
    "followers" : ["1", "3"]
}

PUT /tweets/tweet/1
{
    "user" : "1"
}

GET /tweets/_search
{
    "query" : {
        "terms" : {
            "user" : {
                "index" : "users",
                "type" : "user",
                "id" : "2",
                "path" : "followers"
            }
        }
    }
}
```

外部词条文档的结构还可以包括一个内部对象的数组，例如：

```
PUT /users/user/2
{
 "followers" : [
   {
     "id" : "1"
   },
   {
     "id" : "2"
   }
 ]
}
```

在这种情况下，查找路径将是followers.id。

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/5.4/query-dsl-terms-query.html
