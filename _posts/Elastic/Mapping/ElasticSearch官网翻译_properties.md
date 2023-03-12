---
title: "ElasticSearch官网翻译_properties"
date: 2017-05-20 13:19:36
tags: [Elasticsearch]
categories: [Search]
---

#### properties（属性）

<b>
类型映射，object fields（对象字段）和nested fields（嵌套字段）包含的sub-fields（子字段）称为properties（属性）。 这些属性可能是任何数据类型，包括object（对象）和nested（嵌套）。 属性可以添加：

- 显式地通过在创建索引时定义它们。
- 通过在使用PUT mapping API添加或更新映射类型时定义它们。
- 动态地通过索引文档时包含新字段。
</b>

以下是将属性添加到映射类型，object field（对象字段）和nested field（嵌套字段）的示例：

```
PUT my_index
{
  "mappings": {
    "my_type": { (1)
      "properties": {
        "manager": { (2)
          "properties": {
            "age":  { "type": "integer" },
            "name": { "type": "string"  }
          }
        },
        "employees": { (3)
          "type": "nested",
          "properties": {
            "age":  { "type": "integer" },
            "name": { "type": "string"  }
          }
        }
      }
    }
  }
}

PUT my_index/my_type/1 (4)
{
  "region": "US",
  "manager": {
    "name": "Alice White",
    "age": 30
  },
  "employees": [
    {
      "name": "John Smith",
      "age": 34
    },
    {
      "name": "Peter Brown",
      "age": 26
    }
  ]
}
```

- (1) my_type映射类型下的properties属性。
- (2) manager object field（对象字段）下的properties属性。
- (3) employees nested field（嵌套字段）下的properties属性。
- (4) 一个对应于上述映射的示例文档。

###### 提示

properties设置允许在同一索引中具有相同名称的字段的不同设置。 可以使用PUT mapping API将新属性添加到现有字段。

##### 点符号

<b>内部字段可以在查询，聚合等中使用点符号来引用：</b>

```
GET my_index/_search
{
  "query": {
    "match": {
      "manager.name": "Alice White" 
    }
  },
  "aggs": {
    "Employees": {
      "nested": {
        "path": "employees"
      },
      "aggs": {
        "Employee Ages": {
          "histogram": {
            "field": "employees.age", 
            "interval": 5
          }
        }
      }
    }
  }
}
```

###### 重要

<b>必须指定内部字段的完整路径。</b>

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/2.4/properties.html
