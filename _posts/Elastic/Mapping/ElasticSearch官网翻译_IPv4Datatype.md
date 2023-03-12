---
title: "ElasticSearch官网翻译_IPv4Datatype"
date: 2017-05-11 23:44:56
tags: [Elasticsearch]
categories: [Search]
---

#### IPv4 datatype

ip字段实际上是一个long类型字段，它接受IPv4地址并将它们作为long类型的值进行索引：

```
PUT my_index
{
  "mappings": {
    "my_type": {
      "properties": {
        "ip_addr": {
          "type": "ip"
        }
      }
    }
  }
}

PUT my_index/my_type/1
{
  "ip_addr": "192.168.1.1"
}

GET my_index/_search
{
  "query": {
    "range": {
      "ip_addr": {
        "gte": "192.168.1.0",
        "lt":  "192.168.2.0"
      }
    }
  }
}
```

#### ip字段的参数

ip字段接受以下参数：

- boost : Field-level 索引boost权重提升。 接受一个浮点数，默认为1.0。

- doc_values : 该字段是否应该以column-stride多列的方式存储在磁盘上，以便以后可以将其用于排序，聚合或脚本？ 接受true（默认）或false。

- include_in_all : 字段值是否应包含在_all字段中？可以接受true或false。如果index设置为no，或者如果父对象字段将include_in_all设置为false，则默认为false。否则默认为true。

- index : 该字段是否可以被搜索？ 接收not_analyzed（默认）或者no。

- null_value : 接受一个IPv4的值作为替换任何显式null空值的字段。默认为null，这意味着该字段被视为丢失。

- precision_step : 控制索引的额外条件数量，使范围查询更快。默认为16。

- store : 字段值是否应与_source字段分开存储和检索。 接受true或false（默认）。

注意：IPv6地址目前还不支持

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/2.4/ip.html