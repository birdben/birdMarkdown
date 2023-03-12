---
title: "ElasticSearch官网翻译_CountAPI"
date: 2017-05-28 11:47:31
tags: [Elasticsearch]
categories: [Search]
---

#### Count API

Count API允许轻松执行查询并获取该查询的匹配数。 它可以跨越一个或多个索引并跨越一个或多个类型执行。 可以使用简单query string查询字符串作为参数或使用在请求体内定义的Query DSL来提供查询。 这是一个例子：

```
$ curl -XGET 'http://localhost:9200/twitter/tweet/_count?q=user:kimchy'

$ curl -XGET 'http://localhost:9200/twitter/tweet/_count' -d '
{
    "query" : {
        "term" : { "user" : "kimchy" }
    }
}'
```

###### 注意

在body中发送的查询必须嵌套在query key中，与search API相同

上面的两个例子都做同样的事情，这是一个用户的twitter索引的tweet数量。 结果是：

```
{
    "count" : 1,
    "_shards" : {
        "total" : 5,
        "successful" : 5,
        "failed" : 0
    }
}
```

query是可选的，如果没有提供，它将使用match_all来计算所有文档。

##### Multi index, Multi type

Count API可以应用于multiple types in multiple indices。

##### Request Parameters

当使用查询参数q执行count时，传递的查询是使用Lucene查询解析器的查询字符串。 还有其他可以传递的参数：

Name|Description
---|---
df|在查询中未定义字段前缀时使用的默认字段。
analyzer|分析查询字符串时使用的分析器名称。
default_operator|要使用的默认运算符可以是AND或OR。 默认为OR。
lenient|如果设置为true，将会导致基于格式的失败（例如提供文本到数字字段）被忽略。 默认为false。
lowercase_expanded_terms|词条是否应该自动小写化。 默认为true。
analyze_wildcard|应该分析通配符和前缀查询。 默认为false。
terminate_after|每个分片的最大count，到达该数值查询执行将提前终止。 如果设置，响应将有一个boolean字段terminate_early来指示查询执行是否实际已终止。 默认为不使用terminate_after。

##### Request Body

Count可以使用其body内的Query DSL来表达应该执行的查询。 Body内容也可以作为名为source的REST参数传递。

HTTP GET和HTTP POST都可以用于使用body执行count。 由于并非所有客户端都支持GET，所以POST也是允许的。

##### Distributed

Count操作被广播到所有分片上。 对于每个分片id组，选择一个副本并对其执行。 这意味着副本增加了count的可扩展性。

##### Routing

可以指定routing（路由值的逗号分隔列表），以控制将执行count请求的分片。

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/2.4/search-count.html

