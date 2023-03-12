---
title: "ElasticSearch官网翻译_index"
date: 2017-05-19 15:54:20
tags: [Elasticsearch]
categories: [Search]
---

#### index（索引）

索引选项控制字段值如何索引，因此如何搜索它们。 它接受三个值：

- no

不会将此字段值添加到索引。 使用此设置，该字段将不可查询。

- not_analyzed

将字段值添加到索引不变，作为单个term（词条）。 除字符串字段之外，这是支持此选项的所有字段的默认值。 not_analyzed字段通常用于结构化搜索的term-level查询。

- analyzed

此选项仅适用于字符串字段，并且是字符串字段的默认值。 首先分析字符串字段值以将字符串转换为term词条（例如单个字词的列表），然后对其进行索引。 在搜索时，查询字符串通过（通常）相同的分析器生成与索引中相同格式的term（词条）。 正是这个过程启用全文搜索。

例如，你可以使用以下命令创建一个not_analyzed字符串字段：

```
PUT /my_index
{
  "mappings": {
    "my_type": {
      "properties": {
        "status_code": {
          "type": "string",
          "index": "not_analyzed"
        }
      }
    }
  }
}
```

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/2.4/mapping-index.html
