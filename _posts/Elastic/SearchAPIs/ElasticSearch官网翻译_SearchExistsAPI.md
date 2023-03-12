---
title: "ElasticSearch官网翻译_SearchExistsAPI"
date: 2017-05-28 12:10:47
tags: [Elasticsearch]
categories: [Search]
---

#### Search Exists API

###### 警告

在2.1.0中弃用。
使用大小设置为0的常规_search和terminate_after设置为1

Exsits API允许容易地确定提供的查询是否存在任何匹配的文档。 它可以跨越一个或多个索引并跨越一个或多个类型执行。 可以使用简单query string查询字符串作为参数或使用在请求体内定义的Query DSL来提供查询。 这是一个例子：

```
$ curl -XGET 'http://localhost:9200/twitter/tweet/_search/exists?q=user:kimchy'

$ curl -XGET 'http://localhost:9200/twitter/tweet/_search/exists' -d '
{
    "query" : {
        "term" : { "user" : "kimchy" }
    }
}'
```

###### 注意

在body中发送的查询必须嵌套在query key中，与search api的工作方式相同。

上述两个例子都做同样的事情，这是确定某个用户的twitter索引中存在的tweets。 响应主体将采用以下格式：

```
{
    "exists" : true
}
```

##### Multi index, Multi type

Exsits API可以应用于multiple types in multiple indices。

##### Request Parameters

当使用查询参数q执行时，传递的查询是使用Lucene查询解析器的查询字符串。 还有其他可以传递的参数：

Name|Description
---|---
df|在查询中未定义字段前缀时使用的默认字段。
analyzer|分析查询字符串时使用的分析器名称。
default_operator|要使用的默认运算符可以是AND或OR。 默认为OR。
lenient|如果设置为true，将会导致基于格式的失败（例如提供文本到数字字段）被忽略。 默认为false。
lowercase_expanded_terms|词条是否应该自动小写化。 默认为true。
analyze_wildcard|应该分析通配符和前缀查询。 默认为false。

##### Request Body

Exsits可以使用其body内的Query DSL来表达应该执行的查询。 Body内容也可以作为名为source的REST参数传递。

HTTP GET和HTTP POST都可以用于使用body执行count。 由于并非所有客户端都支持GET，所以POST也是允许的。

##### Distributed

Exists操作被广播到所有分片上。 对于每个分片id组，选择一个副本并对其执行。 这意味着副本增加了exists的可扩展性。一旦第一个分片报告匹配的文档存在，存在的操作也会提前终止分片请求。

##### Routing

可以指定routing（路由值的逗号分隔列表），以控制将执行exists请求的分片。

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/2.4/search-exists.html

