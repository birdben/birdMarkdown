---
title: "ElasticSearch官网翻译_ArrayDatatype"
date: 2017-05-10 23:40:38
tags: [Elasticsearch]
categories: [Search]
---

#### Array datatype

<b>在Elasticsearch中，没有专门的数组类型。 默认情况下，任何字段都可以包含零个或多个值，但是数组中的所有值必须是相同的数据类型。</b> 例如：

字符串数组：[ "one", "two" ]
整型数组：[ 1, 2 ]
嵌套数组：[ 1，[ 2,3 ]]，它相当于[ 1，2，3 ]
对象数组：[ { "name": "Mary", "age": 12 }, { "name": "John", "age": 10 }]

- 注意：对象数组

<b>对象的数组不能按照你预期的方式工作：你不能独立于数组中的其他对象来查询每个对象。 如果需要这样做，那么你应该使用nested数据类型而不是object数据类型。</b>

<b>动态添加字段时，数组中的第一个值决定了字段类型。 所有后续值必须是相同的数据类型，或至少可以将后续值强制转换为相同的数据类型。

不支持使用混合数据类型的数组：[10，"some string"]

数组可能包含null空值，它们被配置的null_value替换或完全跳过。 空数组[]被视为缺少的字段 - 没有值的字段。</b>

没有什么需要预配置为了在文档中使用数组，它们是开箱即用的支持：

```
PUT my_index/my_type/1
{
  "message": "some arrays in this document...",
  "tags":  [ "elasticsearch", "wow" ], (1)
  "lists": [ (2)
    {
      "name": "prog_list",
      "description": "programming list"
    },
    {
      "name": "cool_list",
      "description": "cool stuff list"
    }
  ]
}

PUT my_index/my_type/2 (3)
{
  "message": "no arrays in this document...",
  "tags":  "elasticsearch",
  "lists": {
    "name": "prog_list",
    "description": "programming list"
  }
}

GET my_index/_search
{
  "query": {
    "match": {
      "tags": "elasticsearch" (4)
    }
  }
}
```

- (1) <b>tags字段被动态添加为string字段。</b>
- (2) <b>list字段被动态添加为object字段。</b>
- (3) <b>第二个文档不包含数组，但可以被索引到相同的字段。</b>
- (4) 该查询在tags字段中查找elasticsearch，并匹配两个文档。

#### Multi-value字段和倒排索引

所有字段类型都支持Multi-value字段的事实是源于Lucene的结果。 Lucene被设计为全文搜索引擎。 为了能够在大块文本中搜索单个单词，Lucene将文本标记为单个术语，并将每个单词分别添加到倒排索引中。

这意味着即使一个简单的文本字段也必须能够默认支持多个值。 当添加其他数据类型时，例如数字和日期，它们使用与字符串相同的数据结构，因此可以获得多个值。


参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/2.4/array.html