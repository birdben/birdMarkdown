---
title: "ElasticSearch官网翻译_BinaryDatatype"
date: 2017-05-11 00:12:53
tags: [Elasticsearch]
categories: [Search]
---

#### Binary datatype

二进制类型接受二进制值作为Base64编码字符串。 该字段默认情况下不存储，不可搜索：

```
PUT my_index
{
  "mappings": {
    "my_type": {
      "properties": {
        "name": {
          "type": "string"
        },
        "blob": {
          "type": "binary"
        }
      }
    }
  }
}

PUT my_index/my_type/1
{
  "name": "Some binary blob",
  "blob": "U29tZSBiaW5hcnkgYmxvYg==" (1)
}
```

- (1) Base64编码的二进制值不能有嵌入的换行符\n。

#### 二进制字段的参数

以下参数被二进制字段所接受：

- doc_values

该字段是否应该以column-stride多列的方式存储在磁盘上，以便以后可以将其用于排序，聚合或脚本？ 接受true（默认）或false。

- store

字段值是否应与_source字段分开存储和检索。 接受true或false（默认）。


参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/2.4/binary.html