---
title: "ElasticSearch官网翻译_ignoreAbove"
date: 2017-05-18 20:23:33
tags: [Elasticsearch]
categories: [Search]
---

#### ignore_above（忽略超出长度的数据）

超过ignore_above设置的字符串不会被分析器处理，不会被索引。 这主要用于not_analyzed字符串字段，通常用于过滤，聚合和排序。 这些是结构化字段，通常这些字段中的索引很长的term（词条）通常是没有意义的。

```
PUT my_index
{
  "mappings": {
    "my_type": {
      "properties": {
        "message": {
          "type": "string",
          "index": "not_analyzed",
          "ignore_above": 20 (1)
        }
      }
    }
  }
}

PUT my_index/my_type/1 (2)
{
  "message": "Syntax error"
}

PUT my_index/my_type/2 (3)
{
  "message": "Syntax error with some long stacktrace"
}

GET _search (4)
{
  "aggs": {
    "messages": {
      "terms": {
        "field": "message"
      }
    }
  }
}
```

- (1) 该字段将忽略超过20个字符的任何字符串。
- (2) 此文档已成功索引。
- (3) 该文档将被索引，但没有索引message字段。
- (4) 搜索返回两个文档，但只有第一个存在于terms聚合中。

###### 提示

ignore_above设置允许在相同索引的相同名称的字段有不同的配置。 可以使用PUT mapping API在现有字段上更新其值。

此选项对于防止Lucene术语字节长度限制为32766也是有用的。

###### 注意

ignore_above的值是字符数，但Lucene计数字节。 如果使用具有许多非ASCII字符的UTF-8文本，则可能需要将限制设置为32766/3 = 10922，因为UTF-8字符可能占用最多3个字节。

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/2.4/ignore-above.html
