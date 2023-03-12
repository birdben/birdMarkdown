---
title: "ElasticSearch官网翻译_NestedDatatype"
date: 2017-05-11 23:56:31
tags: [Elasticsearch]
categories: [Search]
---

#### Nested datatype

nested嵌套类型是object数据类型的专用版本，允许对象数组可以独立地进行索引和查询。

##### 数组中的对象如何被平坦化

数组中内部对象的字段不能像你所期望的那样工作。 Lucene没有内部对象的概念，所以Elasticsearch将对象层次结构变成一个简单的字段名称和值的列表。 例如，下面的文档：

```
PUT my_index/my_type/1
{
  "group" : "fans",
  "user" : [ (1)
    {
      "first" : "John",
      "last" :  "Smith"
    },
    {
      "first" : "Alice",
      "last" :  "White"
    }
  ]
}
```

- (1) user字段被动态添加为object类型的字段。

内部将转换成一个看起来更像这样的文档：

```
{
  "group" :        "fans",
  "user.first" : [ "alice", "john" ],
  "user.last" :  [ "smith", "white" ]
}
```

<b>user.first和user.last字段被平铺成multi-value字段，并且alice和white之间的关联丢失。 本文档将错误地匹配alice和smith的查询：</b>

```
GET my_index/_search
{
  "query": {
    "bool": {
      "must": [
        { "match": { "user.first": "Alice" }},
        { "match": { "user.last":  "Smith" }}
      ]
    }
  }
}
```

##### 对对象数组使用nested嵌套字段

<b>如果需要索引对象数组并维护数组中每个对象的独立性，则应使用nested嵌套数据类型而不是object对象数据类型。 在内部，嵌套对象将数组中的每个对象作为单独的隐藏文档进行索引，这意味着每个嵌套对象都可以使用nested query（嵌套查询）独立于其他对象进行查询：</b>

```
PUT my_index
{
  "mappings": {
    "my_type": {
      "properties": {
        "user": {
          "type": "nested" (1)
        }
      }
    }
  }
}

PUT my_index/my_type/1
{
  "group" : "fans",
  "user" : [
    {
      "first" : "John",
      "last" :  "Smith"
    },
    {
      "first" : "Alice",
      "last" :  "White"
    }
  ]
}

GET my_index/_search
{
  "query": {
    "nested": {
      "path": "user",
      "query": {
        "bool": {
          "must": [
            { "match": { "user.first": "Alice" }},
            { "match": { "user.last":  "Smith" }} (2)
          ]
        }
      }
    }
  }
}

GET my_index/_search
{
  "query": {
    "nested": {
      "path": "user",
      "query": {
        "bool": {
          "must": [
            { "match": { "user.first": "Alice" }},
            { "match": { "user.last":  "White" }} (3)
          ]
        }
      },
      "inner_hits": { (4)
        "highlight": {
          "fields": {
            "user.first": {}
          }
        }
      }
    }
  }
}
```

- (1) user字段被映射为nested嵌套类型，而不是object类型。
- (2) 此查询不匹配，因为Alice和Smith不在同一个嵌套对象中。
- (3) 此查询匹配，因为Alice和White位于相同的嵌套对象中。
- (4) inner_hits允许我们高亮显示匹配的嵌套文档。

嵌套文件可以是：

- 使用nested query嵌套查询进行查询。
- 使用nested和reverse_nested聚合进行分析。
- 使用nested sorting嵌套排序进行排序。
- 通过nested inner hits进行取回和高亮显示。

##### nested嵌套字段的参数

嵌套字段接受以下参数：

- dynamic : 是否应将新属性动态添加到现有的嵌套对象。 接受true（默认），false和strict。

- include_in_all : 为嵌套对象中的所有属性设置默认的include_in_all值。 嵌套文档没有自己的_all字段。 而是将值添加到主"root"文档的_all字段。

- properties : 嵌套对象中的字段，可以是任何数据类型，包括nested嵌套类型。 新的属性可能会添加到现有的嵌套对象。

###### 重要

<b>
由于嵌套文档作为单独的文档进行索引，因此只能在nested query（嵌套查询），nested（嵌套）/reverse_nested（反向嵌套）或nested inner hits（嵌套内部命中）的范围内进行访问。

例如，如果嵌套文档中的字符串字段的index_options设置为offset（偏移量）以允许使用高亮显示，那么这些偏移量在主要高亮显示阶段将不可用。 相反，高亮显示需要通过嵌套的内部命中来执行。
</b>

##### 限制嵌套字段的数量

<b>
每个嵌套文档作为单独的文档进行索引时，使用100个嵌套字段索引文档实际上会索引101个文档。 为了防止不明确的映射，每个索引可以定义的嵌套字段的数量已被限制为50.默认限制可以通过索引设置index.mapping.nested_fields.limit进行更改。
<b>


参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/2.4/nested.html