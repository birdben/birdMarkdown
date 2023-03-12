---
title: "ElasticSearch官网翻译_copy_to"
date: 2017-05-13 14:30:02
tags: [Elasticsearch]
categories: [Search]
---

#### copy_to（复制合并）

copy_to参数允许你创建自定义_all字段。 换句话说，可以将多个字段的值复制到组字段中，然后可以将其作为单个字段进行查询。 例如，first_name和last_name字段可以复制到full_name字段，如下所示：

```
PUT /my_index
{
  "mappings": {
    "my_type": {
      "properties": {
        "first_name": {
          "type": "string",
          "copy_to": "full_name" (1)
        },
        "last_name": {
          "type": "string",
          "copy_to": "full_name" (2)
        },
        "full_name": {
          "type": "string"
        }
      }
    }
  }
}

PUT /my_index/my_type/1
{
  "first_name": "John",
  "last_name": "Smith"
}

GET /my_index/_search
{
  "query": {
    "match": {
      "full_name": { (3)
        "query": "John Smith",
        "operator": "and"
      }
    }
  }
}
```

- (1)(2) first_name和last_name字段的值将复制到full_name字段。
- (3) first_name和last_name字段仍然可以分别查询first name和last name，但是也可以通过full_name字段查询first name和last name。

<b>

一些要点：

- 它是被复制的字段值，而不是词条（由分析过程产生的结果）。
- 原始_source字段将不被修改以显示复制的值。
- 相同的值可以复制到多个字段，可以使用"copy_to": [ "field_1", "field_2" ]
</b>

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/2.4/copy-to.html
