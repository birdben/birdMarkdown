---
title: "ElasticSearch官网翻译_dynamic"
date: 2017-05-13 15:45:02
tags: [Elasticsearch]
categories: [Search]
---

#### dynamic（动态设置）

默认情况下，可以通过索引一个包含新字段的文档来将字段动态添加到文档或文档中的内部对象。 例如：

```
DELETE my_index (1)

PUT my_index/my_type/1 (2)
{
  "username": "johnsmith",
  "name": {
    "first": "John",
    "last": "Smith"
  }
}

GET my_index/_mapping (3)

PUT my_index/my_type/2 (4)
{
  "username": "marywhite",
  "email": "mary@white.com",
  "name": {
    "first": "Mary",
    "middle": "Alice",
    "last": "White"
  }
}

GET my_index/_mapping (5)
```

- (1) 首先删除索引，以防它已经存在。
- (2) 本文档介绍了，字符串字段username，对象字段name和在name对象下的两个字符串字段，可以对照为name.first和name.last。
- (3) 检查映射以验证上面的结果。
- (4) 此文档添加了两个字符串字段：email和name.middle。
- (5) 检查映射以验证更改。

<b>
Dynamic mapping（动态映射）说明了如何在映射中检测和添加新字段的细节。

dynamic（动态设置）控制是否可以动态添加新字段。 它接受三个设置：

- true : 新检测的字段将添加到映射中。 （默认）
- fase : 新检测的字段将被忽略。 必须明确添加新字段。
- strict : 如果检测到新字段，将抛出异常并拒绝文档。

可以在映射类型级别和每个内部对象上设置动态设置。 内部对象从其父对象或映射类型继承设置。 例如：
</b>

```
PUT my_index
{
  "mappings": {
    "my_type": {
      "dynamic": false, (1)
      "properties": {
        "user": { (2)
          "properties": {
            "name": {
              "type": "string"
            },
            "social_networks": { (3)
              "dynamic": true,
              "properties": {}
            }
          }
        }
      }
    }
  }
}
```

<b>

- (1) 在类型级别禁用Dynamic mapping（动态映射），因此不会动态添加新的顶级字段。
- (2) user对象继承类型级别的设置。
- (3) user.social_networks对象启用Dynamic mapping（动态映射），因此可以将新字段添加到此内部对象。
</b>

###### 提示

Dynamic mapping（动态设置）允许在同一索引中具有相同名称的字段的不同设置。 可以使用PUT mapping API在现有字段上更新其值。


参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/2.4/dynamic.html
