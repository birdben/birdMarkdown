---
title: "ElasticSearch官网翻译_Search"
date: 2017-05-23 18:10:57
tags: [Elasticsearch]
categories: [Search]
---

#### Search

搜索API允许你执行搜索查询并获取与查询匹配的搜索匹配。 可以使用简单的query string as a parameter（查询字符串作为参数）或使用request body来提供查询。

##### Multi-Index, Multi-Type

所有搜索API可以应用于跨索引中的多种类型，并跨多个索引并支持multi index syntax（多索引语法）。 例如，我们可以搜索twitter索引中所有类型的所有文档：

```
$ curl -XGET 'http://localhost:9200/twitter/_search?q=user:kimchy'
```

我们也可以搜索具体类型：

```
$ curl -XGET 'http://localhost:9200/twitter/tweet,user/_search?q=user:kimchy'
```

我们还可以通过多个索引搜索带有某个标签的所有tweet（类型）（例如，每个用户都有自己的索引）：

```
$ curl -XGET 'http://localhost:9200/kimchy,elasticsearch/tweet/_search?q=tag:wow'
```

或者我们可以使用_all占位符搜索所有可用索引的所有tweet（类型）：

```
$ curl -XGET 'http://localhost:9200/_all/tweet/_search?q=tag:wow'
```

甚至搜索所有索引和所有类型：

```
$ curl -XGET 'http://localhost:9200/_search?q=tag:wow'
```

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/2.4/search-search.html

