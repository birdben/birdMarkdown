---
title: "ElasticSearch官网翻译_DateDatatype"
date: 2017-05-11 14:31:04
tags: [Elasticsearch]
categories: [Search]
---

#### Date datatype

<b>
注意：以下文中所提到的名词解释：

- "毫秒数"：都是指从1970年1月1日00:00:00 UTC开始的毫秒数
- "秒数"：都是指从1970年1月1日00:00:00 UTC开始的秒数
</b>

JSON没有日期数据类型，所以Elasticsearch中的日期可以是：

- 包含格式化日期的字符串，例如："2015-01-01"或"2015/01/01 12:10:30"。
- 一个long类型数字代表了"毫秒数"。
- 一个integer类型数字代表了"秒数"。

<b>在内部，日期将转换为UTC（如果指定了时区），并将其存储为表示"毫秒数"。</b>

日期格式可以自定义，但如果没有指定格式，则使用默认格式：

```
"strict_date_optional_time|| epoch_millis"
```

这意味着它将接受带有可选时间戳的日期，这些日期符合strict_date_optional_time或者"毫秒数"的格式。

举例：

```
PUT my_index
{
  "mappings": {
    "my_type": {
      "properties": {
        "date": {
          "type": "date" (1)
        }
      }
    }
  }
}

PUT my_index/my_type/1
{ "date": "2015-01-01" } (2)

PUT my_index/my_type/2
{ "date": "2015-01-01T12:10:30Z" } (3)

PUT my_index/my_type/3
{ "date": 1420070400001 } (4)

GET my_index/_search
{
  "sort": { "date": "asc"} (5)
}
```

- (1) date字段使用默认格式。
- (2) 这个文档使用纯文本的日期格式。
- (3) 这个文档包含了时间。
- (4) 这个文档使用了"毫秒数"的格式。
- (5) 请注意，返回的排序值全部以"毫秒数"为单位。


#### date 字段的参数

日期字段接受以下参数：

- boost : Field-level 索引boost权重提升。 接受一个浮点数，默认为1.0。

- doc_values : 该字段是否应该以column-stride多列的方式存储在磁盘上，以便以后可以将其用于排序，聚合或脚本？ 接受true（默认）或false。

- format : 可以解析的日期格式。默认为strict_date_optional_time || epoch_millis。

- ignore_malformed : 如果为true，则格式错误的数字将被忽略。如果为false（默认），格式错误的数字会引发异常并拒绝整个文档。

- include_in_all : 字段值是否应包含在_all字段中？可以接受true或false。如果index设置为no，或者如果父对象字段将include_in_all设置为false，则默认为false。否则默认为true。

- index : 该字段是否可以被搜索？ 接收not_analyzed（默认）或者no。

- null_value : 接受一个配置格式的日期值作为替换任何显式null空值的字段。默认为null，这意味着该字段被视为丢失。

- precision_step : 控制索引的额外条件数量，使范围查询更快。默认为16。

- store : 字段值是否应与_source字段分开存储和检索。 接受true或false（默认）。


参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/2.4/date.html
