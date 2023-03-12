---
title: "ElasticSearch官网翻译_SourceFiltering"
date: 2017-05-24 21:22:52
tags: [Elasticsearch]
categories: [Search]
---

#### Source filtering

允许控制每次命中返回_source字段的方式。

默认情况下，操作将返回_source字段的内容，除非你使用fields参数或者_source字段被禁用。

你可以使用_source参数关闭_source取回：

要禁用_source取回设置为false：

```
{
    "_source": false,
    "query" : {
        "term" : { "user" : "kimchy" }
    }
}
```

<b>_source还接受一个或多个通配符模式来控制_source的哪些部分应该返回：</b>

例如：

```
{
    "_source": "obj.*",
    "query" : {
        "term" : { "user" : "kimchy" }
    }
}
```

或者

```
{
    "_source": [ "obj1.*", "obj2.*" ],
    "query" : {
        "term" : { "user" : "kimchy" }
    }
}
```

<b>最后，为了完全控制，你可以指定包含和排除模式：</b>

```
{
    "_source": {
        "include": [ "obj1.*", "obj2.*" ],
        "exclude": [ "*.description" ]
    },
    "query" : {
        "term" : { "user" : "kimchy" }
    }
}
```

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/2.4/search-request-source-filtering.html

