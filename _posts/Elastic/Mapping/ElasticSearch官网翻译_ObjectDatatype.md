---
title: "ElasticSearch官网翻译_ObjectDatatype"
date: 2017-05-12 01:33:48
tags: [Elasticsearch]
categories: [Search]
---

#### Object datatypes

JSON文档本质上是分层的：文档可能包含内部对象，而内部对象又可能包含内部对象本身：

```
PUT my_index/my_type/1
{ (1)
  "region": "US",
  "manager": { (2)
    "age":     30,
    "name": { (3)
      "first": "John",
      "last":  "Smith"
    }
  }
}
```

- (1) 外部文档也是一个JSON对象。
- (2) 它包含一个名字为manager的内部对象。
- (3) 它又包含一个名字为name的内部对象。

在内部，该文档被索引为一个简单的，扁平的键值对列表，如下所示：

```
{
  "region":             "US",
  "manager.age":        30,
  "manager.name.first": "John",
  "manager.name.last":  "Smith"
}
```

上述文档的显式映射可能如下所示：

```
PUT my_index
{
  "mappings": {
    "my_type": { (1)
      "properties": {
        "region": {
          "type": "string",
          "index": "not_analyzed"
        },
        "manager": { (2)
          "properties": {
            "age":  { "type": "integer" },
            "name": { (3)
              "properties": {
                "first": { "type": "string" },
                "last":  { "type": "string" }
              }
            }
          }
        }
      }
    }
  }
}
```

- (1) 映射类型是一个对象类型，并具有一个properties（属性）字段。
- (2) manager字段是内部object（对象）字段。
- (3) manager.name字段是manager字段内的一个内部object（对象）字段。

<b>你不需要将字段type（类型）显式设置为object（对象），因为这是默认值。</b>

##### 对象字段的参数

对象字段接受以下参数：

- dynamic : 是否应将新属性动态添加到现有的对象。 接受true（默认），false和strict。

- enabled : 是否应该对该对象字段给出的JSON值进行解析和索引（true，default）或完全忽略（false）。

- include_in_all : 为嵌套对象中的所有属性设置默认的include_in_all值。 嵌套文档没有自己的_all字段。 而是将值添加到主"root"文档的_all字段。

- <b>properties : 对象内的字段，可以是任何数据类型，包括object对象类型。 可以将新属性添加到现有对象。</b>

###### 重要

如果需要索引对象数组而不是单个对象，请先阅读Nested嵌套数据类型。

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/2.4/object.html