---
title: "ElasticSearch官网翻译_TokenCountDatatype"
date: 2017-05-12 02:49:09
tags: [Elasticsearch]
categories: [Search]
---

#### Token Count datatypes

类型为token_count的字段实际上是一个接受字符串值的integer整数字段，对它们进行分析，然后对字符串中的词元数进行索引。

举例：

```
PUT my_index
{
  "mappings": {
    "my_type": {
      "properties": {
        "name": { (1)
          "type": "string",
          "fields": {
            "length": { (2)
              "type":     "token_count",
              "analyzer": "standard"
            }
          }
        }
      }
    }
  }
}

PUT my_index/my_type/1
{ "name": "John Smith" }

PUT my_index/my_type/2
{ "name": "Rachel Alice Williams" }

GET my_index/_search
{
  "query": {
    "term": {
      "name.length": 3 (3)
    }
  }
}
```

- (1) name字段是使用默认standard analyzer标准分析器的分析字符串字段。
- (2) name.length字段是一个token_count多字段，它将索引词元的数量到name字段中。
- (3) 此查询仅匹配包含Rachel Alice Williams的文档，因为它包含三个词元。

###### 注意

<b>在技术上，token_count类型对位置增量进行求和，而不是计数词元。 这意味着即使分析器过滤掉stop word（停词），它们也包括在计数中。</b>

#### token_count字段的参数

token_count字段接受以下参数：

- analyzer : analyzer（分析器）应用于analyzed（分析的）字符串字段。为获得最佳性能，请使用不带词元过滤器的分析器。

- boost : Field-level 索引boost权重提升。 接受一个浮点数，默认为1.0。

- doc_values : 该字段是否应该以column-stride多列的方式存储在磁盘上，以便以后可以将其用于排序，聚合或脚本？ 接受true（默认）或false。

- index : 该字段是否可以被搜索？ 接收not_analyzed（默认）或者no。

- include_in_all : 字段值是否应包含在_all字段中？可以接受true或false。如果index设置为no，或者如果父对象字段将include_in_all设置为false，则默认为false。否则默认为true。

- null_value : 接受一个数字值作为替换任何显式null空值的字段。默认为null，这意味着该字段被视为丢失。

- precision_step : 控制索引的额外条件数量，使范围查询更快。默认为32。

- store : 字段值是否应与_source字段分开存储和检索。 接受true或false（默认）。

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/2.4/token-count.html
