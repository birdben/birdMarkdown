---
title: "ElasticSearch官网翻译_enabled"
date: 2017-05-13 16:09:16
tags: [Elasticsearch]
categories: [Search]
---

#### enabled（开启字段）

Elasticsearch尝试索引所有你提供的字段，但有时你只想存储该字段而不对其进行索引。 例如，假设你正在使用Elasticsearch作为Web会话存储。 你可能需要索引会话ID和上次更新时间，但不需要在会话数据本身上查询或运行聚合。

enable设置只能应用于映射类型和对象字段，导致Elasticsearch完全跳过对该字段内容的解析。 JSON仍然可以从_source字段检索，但是不能以任何其他方式搜索或存储：

```
PUT my_index
{
  "mappings": {
    "session": {
      "properties": {
        "user_id": {
          "type":  "string",
          "index": "not_analyzed"
        },
        "last_updated": {
          "type": "date"
        },
        "session_data": { 
          "enabled": false
        }
      }
    }
  }
}

PUT my_index/session/session_1
{
  "user_id": "kimchy",
  "session_data": { 
    "arbitrary_object": {
      "some_array": [ "foo", "bar", { "baz": 2 } ]
    }
  },
  "last_updated": "2015-12-06T18:20:22"
}

PUT my_index/session/session_2
{
  "user_id": "jpountz",
  "session_data": "none", 
  "last_updated": "2015-12-06T18:22:13"
}
```

- (1) session_data字段被禁用。
- (2) 任何数据都可以被传递到session_data字段，因为它将被完全忽略。
- (3) session_data还将忽略不是JSON对象的值。

整个映射类型也可以被禁用，在这种情况下，文档存储在_source字段中，这意味着它可以被检索，但是它的内容没有以任何方式被索引：

```
PUT my_index
{
  "mappings": {
    "session": { (1)
      "enabled": false
    }
  }
}

PUT my_index/session/session_1
{
  "user_id": "kimchy",
  "session_data": {
    "arbitrary_object": {
      "some_array": [ "foo", "bar", { "baz": 2 } ]
    }
  },
  "last_updated": "2015-12-06T18:20:22"
}

GET my_index/session/session_1 (2)

GET my_index/_mapping (3)
```

- (1) 整个session映射类型被禁用。
- (2) 该文档可以被检索。
- (3) 检查映射显示没有添加字段。

###### 提示

enabled设置允许在同一索引中具有相同名称的字段的不同设置。 可以使用PUT mapping API在现有字段上更新其值。


参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/2.4/enabled.html
