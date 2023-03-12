---
title: "ElasticSearch官网翻译_ignoreMalformed"
date: 2017-05-19 01:41:22
tags: [Elasticsearch]
categories: [Search]
---

#### ignore_malformed（忽略格式不正确的数据）

有时你对你收到的数据没有太多的控制权。 一个用户可以发送日期作为login字段，另一个用户发送电子邮件地址作为login字段。

尝试将错误的数据类型索引到字段中，默认情况下会引发异常，并拒绝整个文档。 ignore_malformed参数（如果设置为true）允许忽略该异常。 格式不正确的字段不会被索引，但文档中的其他字段正常处理。

例如：

```
PUT my_index
{
  "mappings": {
    "my_type": {
      "properties": {
        "number_one": {
          "type": "integer"
        },
        "number_two": {
          "type": "integer",
          "ignore_malformed": true
        }
      }
    }
  }
}

PUT my_index/my_type/1
{
  "text":       "Some text value",
  "number_one": "foo" (1)
}

PUT my_index/my_type/2
{
  "text":       "Some text value",
  "number_two": "foo" (2)
}
```

- (1) 此文档将被拒绝，因为number_one不允许格式错误的值。
- (2) 该文档text字段会被索引，而number_two字段不会被索引。

###### 提示

ignore_malformed设置允许在相同索引中相同名称的字段有不同的设置。 可以使用PUT mapping API在现有字段上更新其值。

##### 索引级默认

可以在索引级别上设置index.mapping.ignore_malformed设置，以允许在所有映射类型中全局忽略格式错误的内容。

```
PUT my_index
{
  "settings": {
    "index.mapping.ignore_malformed": true (1)
  },
  "mappings": {
    "my_type": {
      "properties": {
        "number_one": { (2)
          "type": "byte"
        },
        "number_two": {
          "type": "integer",
          "ignore_malformed": false (3)
        }
      }
    }
  }
}
```

- (1)(2) number_one字段继承索引级设置。
- (3) number_two字段覆盖索引级别设置以关闭ignore_malformed。

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/2.4/ignore-malformed.html
