---
title: "ElasticSearch官网翻译_NumericDatatype"
date: 2017-05-12 00:44:32
tags: [Elasticsearch]
categories: [Search]
---

#### Numeric datatypes

支持以下数字类型：

- long : 带符号的64位整数，最小值为-2<sup>63</sup>，最大值为2<sup>63-1</sup>。
- integer : 一个带符号的32位整数，最小值为-2<sup>31</sup>，最大值为2<sup>31-1</sup>。
- short : 一个带符号的16位整数，最小值为-32,768，最大值为32,767。
- byte : 一个带符号的8位整数，最小值为-128，最大值为127。
- double : 双精度64位IEEE 754浮点数。
- float : 单精度32位IEEE 754浮点数。

以下是使用numeric字段配置映射的示例：

```
PUT my_index
{
  "mappings": {
    "my_type": {
      "properties": {
        "number_of_bytes": {
          "type": "integer"
        },
        "time_in_seconds": {
          "type": "float"
        }
      }
    }
  }
}
```

##### 数字字段的参数

数字类型接受以下参数：

- coerce : 尝试将字符串转换为数字并截断整数的部分。接受true（默认）和false。
- boost : Field-level 索引boost权重提升。 接受一个浮点数，默认为1.0。
- doc_values : 该字段是否应该以column-stride多列的方式存储在磁盘上，以便以后可以将其用于排序，聚合或脚本？ 接受true（默认）或false。
- ignore_malformed : 如果为true，则格式错误的数字将被忽略。如果为false（默认），格式错误的数字会引发异常并拒绝整个文档。
- include_in_all : 字段值是否应包含在_all字段中？可以接受true或false。如果index设置为no，或者如果父对象字段将include_in_all设置为false，则默认为false。否则默认为true。
- index : 该字段是否可以被搜索？ 接收not_analyzed（默认）或者no。
- null_value : 接受一个数字值作为替换任何显式null空值的字段。默认为null，这意味着该字段被视为丢失。
- precision_step : 控制索引的额外条件数量，使范围查询更快。默认为16。
- store : 字段值是否应与_source字段分开存储和检索。 接受true或false（默认）。


参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/2.4/number.html