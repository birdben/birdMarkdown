---
title: "ElasticSearch官网翻译_coerce"
date: 2017-05-13 14:19:57
tags: [Elasticsearch]
categories: [Search]
---

#### coerce（强制类型转换）

数据并不总是很干净。 根据它的生成方式，一个数字可能会在JSON体中呈现为一个真正的JSON数字，例如 5，但它也可以呈现为字符串，例如："5"。 或者，可以将应该是整数的数字呈现为浮点，例如：5.0或甚至"5.0"。

<b>
Coercion（强制类型转换）尝试清除脏值以适应字段的数据类型。 例如：

- string字符串将被强制转换为number数字类型。
- float浮点将被截断为integer整数值。
- Lon/lat geo-points将被标准化为标准-180:180/-90:90坐标系。
</b>

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
          "coerce": false
        }
      }
    }
  }
}

PUT my_index/my_type/1
{
  "number_one": "10" (1)
}

PUT my_index/my_type/2
{
  "number_two": "10" (2)
}
```

- (1) number_one字段将转换为整数10。
- (2) 该文档将被拒绝，因为coercion（强制类型转换）被禁用。

###### 提示

coerce（强制类型转换）设置允许在相同索引中具有相同名称的字段的不同设置。 可以使用PUT mapping API在现有字段上更新其值。

##### 索引级默认

index.mapping.coerce设置可以在索引级别设置，以在所有映射类型之间全局禁用强制：

```
PUT my_index
{
  "settings": {
    "index.mapping.coerce": false
  },
  "mappings": {
    "my_type": {
      "properties": {
        "number_one": {
          "type": "integer"
        },
        "number_two": {
          "type": "integer",
          "coerce": true
        }
      }
    }
  }
}

PUT my_index/my_type/1
{ "number_one": "10" } (1)

PUT my_index/my_type/2
{ "number_two": "10" } (2)
```

- (1) 该文档将被拒绝，因为number_one字段继承了索引级别的coerce（强制类型转换）设置。
- (2) number_two字段覆盖索引级别设置以启用coerce（强制类型转换）。


参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/2.4/coerce.html
