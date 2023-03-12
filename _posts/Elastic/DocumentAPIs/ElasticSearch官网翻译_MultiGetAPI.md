---
title: "ElasticSearch官网翻译_MultiGetAPI"
date: 2017-06-01 16:23:10
tags: [Elasticsearch]
categories: [Search]
---

## Multi Get API

Multi GET API允许基于索引，类型（可选）和id（以及可能的路由）获取多个文档。 响应包括一个具有所有获取的文档的docs数组，每个元素的结构与get API提供的文档相似。 这是一个例子：

```
GET _mget
{
    "docs" : [
        {
            "_index" : "test",
            "_type" : "type",
            "_id" : "1"
        },
        {
            "_index" : "test",
            "_type" : "type",
            "_id" : "2"
        }
    ]
}
```

mget endpoint也可以用于索引（在这种情况下，它不需要在主体中）：

```
GET test/_mget
{
    "docs" : [
        {
            "_type" : "type",
            "_id" : "1"
        },
        {
            "_type" : "type",
            "_id" : "2"
        }
    ]
}
```

type也是同理：

```
GET test/type/_mget
{
    "docs" : [
        {
            "_id" : "1"
        },
        {
            "_id" : "2"
        }
    ]
}
```

在这种情况下，ids元素可以直接用于简化请求：

```
GET test/type/_mget
{
    "ids" : ["1", "2"]
}
```

### Optional Type（可选的类型）

mget API允许_type是可选的。 将其设置为_all或将其留空，以便获取与所有类型的id匹配的第一个文档。

如果你没有设置类型，并且有很多文档共享相同的_id，你将最终只得到第一个匹配的文档。

例如，如果你在typeA和typeB中有文档1，则以下请求将仅返回相同的文档两次：

```
GET test/_mget
{
    "ids" : ["1", "1"]
}
```

在这种情况下需要明确设置_type：

```
GET test/_mget/
{
  "docs" : [
        {
            "_type":"typeA",
            "_id" : "1"
        },
        {
            "_type":"typeB",
            "_id" : "1"
        }
    ]
}
```

### Source filtering（源过滤器）

默认情况下，将为每个文档（如果存储）返回_source字段。 与get API类似，你可以使用_source参数来只取回_source（或根本不）的部分。 你还可以使用url参数_source，_source_include和_source_exclude来指定默认值，当没有为每个文档特殊指定时，它将被使用。

例如：

```
GET _mget
{
    "docs" : [
        {
            "_index" : "test",
            "_type" : "type",
            "_id" : "1",
            "_source" : false
        },
        {
            "_index" : "test",
            "_type" : "type",
            "_id" : "2",
            "_source" : ["field3", "field4"]
        },
        {
            "_index" : "test",
            "_type" : "type",
            "_id" : "3",
            "_source" : {
                "include": ["user"],
                "exclude": ["user.location"]
            }
        }
    ]
}
```

### Fields（字段）

可以根据Get API的stored_fields参数，指定特定的存储字段以便每个文档取回。 例如：

```
GET _mget
{
    "docs" : [
        {
            "_index" : "test",
            "_type" : "type",
            "_id" : "1",
            "stored_fields" : ["field1", "field2"]
        },
        {
            "_index" : "test",
            "_type" : "type",
            "_id" : "2",
            "stored_fields" : ["field3", "field4"]
        }
    ]
}
```

或者，你可以将查询字符串中的stored_fields参数指定为默认值以应用于所有文档。

```
GET test/type/_mget?stored_fields=field1,field2
{
    "docs" : [
        {
            "_id" : "1" (1)
        },
        {
            "_id" : "2",
            "stored_fields" : ["field3", "field4"] (3)
        }
    ]
}
```

- (1) 返回field1和field2
- (2) 返回field3和field4

### Generated fields（生成字段）

有关仅在索引时生成的字段，请参阅"Generated fields"一节。

### Routing（路由）

你也可以指定routing作为参数：

```
GET _mget?routing=key1
{
    "docs" : [
        {
            "_index" : "test",
            "_type" : "type",
            "_id" : "1",
            "_routing" : "key2"
        },
        {
            "_index" : "test",
            "_type" : "type",
            "_id" : "2"
        }
    ]
}
```

在这个例子中，文档test/type/2将从对应于routing＝key1的分片中获取。但文档test/type/1将从对应于routing＝key1的分片中获取。

### Security

见 URL-based access control 

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/5.4/docs-multi-get.html
