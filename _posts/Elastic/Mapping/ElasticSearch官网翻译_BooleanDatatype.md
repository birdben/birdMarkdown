---
title: "ElasticSearch官网翻译_BooleanDatatype"
date: 2017-05-11 00:19:54
tags: [Elasticsearch]
categories: [Search]
---

#### Boolean datatype

布尔字段接受JSON true和false值，但也可以接受被解释为true或false的字符串和数字：

- False values : false, "false", "off", "no", "0", "" (empty string), 0, 0.0
- True values : 任何不是false的

举例：

```
PUT my_index
{
  "mappings": {
    "my_type": {
      "properties": {
        "is_published": {
          "type": "boolean"
        }
      }
    }
  }
}

POST my_index/my_type/1
{
  "is_published": true (1)
}

GET my_index/_search
{
  "query": {
    "term": {
      "is_published": 1 (2)
    }
  }
}
```

	
- (1) 使用JSON true索引文档
- (2) 使用1查询文档, 将其解释为true

聚合term aggregation使用1和0的key，以及key_as_string的字符串"true"和"false"。 在脚本中使用布尔字段，返回1和0：

```
POST my_index/my_type/1
{
  "is_published": true
}

POST my_index/my_type/2
{
  "is_published": false
}

GET my_index/_search
{
  "aggs": {
    "publish_state": {
      "terms": {
        "field": "is_published"
      }
    }
  },
  "script_fields": {
    "is_published": {
      "script": "doc['is_published'].value" (1)
    }
  }
}
```

- (1) Inline scripts必须设置成enabled这个例子才能正常运行

#### 布尔字段的参数

布尔字段接受以下参数：

- boost : Field-level 索引boost权重提升。 接受一个浮点数，默认为1.0。

- doc_values : 该字段是否应该以column-stride多列的方式存储在磁盘上，以便以后可以将其用于排序，聚合或脚本？ 接受true（默认）或false。

- index : 该字段是否可以被搜索？ 接收not_analyzed（默认）或者no。

- null_value : 接受上面列出的任何true或者false值。该值替换任何显式的null空值。默认为null，这意味着该字段被视为丢失。

- store : 字段值是否应与_source字段分开存储和检索。 接受true或false（默认）。

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/2.4/boolean.html
