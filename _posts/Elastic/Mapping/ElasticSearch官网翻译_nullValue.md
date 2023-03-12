---
title: "ElasticSearch官网翻译_nullValue"
date: 2017-05-20 11:15:49
tags: [Elasticsearch]
categories: [Search]
---

#### null_value（空值）

<b>null空值不能被索引或搜索。 当一个字段设置为null（或一个空数组或一个null空值数组）时，它被视为该字段没有值。</b>

<b>null_value参数允许你使用指定的值替换显式null空值，以便对其进行索引和搜索。</b> 例如：

```
PUT my_index
{
  "mappings": {
    "my_type": {
      "properties": {
        "status_code": {
          "type":       "string",
          "index":      "not_analyzed",
          "null_value": "NULL" 
        }
      }
    }
  }
}

PUT my_index/my_type/1
{
  "status_code": null
}

PUT my_index/my_type/2
{
  "status_code": [] 
}

GET my_index/_search
{
  "query": {
    "term": {
      "status_code": "NULL" 
    }
  }
}
```

- (1) 用"NULL"字符串替换显式null空值。
- (2) <b>一个空数组不包含一个显式的null空值，所以不会被null_value所替代。</b>
- (3) 对"NULL"的查询返回文档1，但不是文档2。

###### 重要

<b>
null_value需要与字段相同的数据类型。 例如，long字段不能有字符串null_value。 analyzed的字符串字段也将通过配置的analyzer分析器分析null_value配置的字符串。
</b>

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/2.4/null-value.html

