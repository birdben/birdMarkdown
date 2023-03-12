---
title: "ElasticSearch官网翻译_PutMapping"
date: 2017-06-03 16:53:30
tags: [Elasticsearch]
categories: [Search]
---

## Put Mapping

PUT mapping API允许你向现有索引添加新类型，或向现有类型添加新字段：

```
PUT twitter (1)
{
  "mappings": {
    "tweet": {
      "properties": {
        "message": {
          "type": "text"
        }
      }
    }
  }
}

PUT twitter/_mapping/user (2)
{
  "properties": {
    "name": {
      "type": "text"
    }
  }
}

PUT twitter/_mapping/tweet (3)
{
  "properties": {
    "user_name": {
      "type": "text"
    }
  }
}
```

- (1) 创建一个名为twitter的索引，并且在tweet映射类型中带有message字段。
- (2) 使用PUT mapping API添加一个名为user的新映射类型。
- (3) 使用PUT mapping API向tweet映射类型添加一个名为user_name的新字段。

有关如何定义类型映射的更多信息，请参见映射部分。

### Multi-index（多索引）

PUT mapping API可以应用于具有单个请求的多个索引。 它具有以下格式：

```
PUT /{index}/_mapping/{type}
{ body }
```

- (1) {index}接受多个索引名称和通配符。
- (2) {type}是要更新的类型的名称。
- (3) {body}包含应该应用的映射更改。

###### 注意

> 当使用PUT mapping API更新_default_映射时，新映射不会与现有映射合并。 相反，新的_default_映射替换现有的_default_映射。

### Updating field mappings（更新字段映射）

一般来说，现有字段的映射无法更新。 这个规则有一些例外。 例如：

- 可以将新properties属性添加到Object数据类型字段中。
- 新的multi-fields多字段可以添加到现有字段。
- 可以更新ignore_above参数。

例如：

```
PUT my_index (1)
{
  "mappings": {
    "user": {
      "properties": {
        "name": {
          "properties": {
            "first": {
              "type": "text"
            }
          }
        },
        "user_id": {
          "type": "keyword"
        }
      }
    }
  }
}

PUT my_index/_mapping/user
{
  "properties": {
    "name": {
      "properties": {
        "last": { (2)
          "type": "text"
        }
      }
    },
    "user_id": {
      "type": "keyword",
      "ignore_above": 100 (3)
    }
  }
}
```

- (1) 创建一个索引带有first字段，并且在名为name的Object datatype字段下，还创建一个user_id字段。
- (2) 添加last字段到名为name的Object datatype字段下。
- (3) 将ignore_above设置从默认值0更新到100。

每个mapping parameter（映射参数）指定是否可以在现有字段上更新其设置。

### Conflicts between fields in different types（字段不同类型冲突）

在同一个索引中具有相同名称的两个不同类型的字段必须具有相同的映射，因为它们在内部由相同的字段支持。 尝试更新存在于多个类型中的字段的映射参数将抛出异常，除非你指定update_all_types参数，在这种情况下将在同一索引中的同一个名称的所有字段上更新该参数。

###### 提示

> 唯一可以免除此规则的参数 - 它们可以设置为每个字段上的不同值 - 可以在"Fields are shared across mapping types"一节中找到。

例如，这会失败：

```
PUT my_index
{
  "mappings": {
    "type_one": {
      "properties": {
        "text": { (1)
          "type": "text",
          "analyzer": "standard"
        }
      }
    },
    "type_two": {
      "properties": {
        "text": { (2)
          "type": "text",
          "analyzer": "standard"
        }
      }
    }
  }
}

PUT my_index/_mapping/type_one (3)
{
  "properties": {
    "text": {
      "type": "text",
      "analyzer": "standard",
      "search_analyzer": "whitespace"
    }
  }
}
```

- (1)(2) 创建一个两种类型的索引，它们都包含一个具有相同映射的text字段。
- (3) 尝试只是为了type_one更新search_analyzer抛出一个类型"Merge failed with failures..."的异常。

但是这样可以运行成功了：

```
PUT my_index/_mapping/type_one?update_all_types (1)
{
  "properties": {
    "text": {
      "type": "text",
      "analyzer": "standard",
      "search_analyzer": "whitespace"
    }
  }
}
```

- (1) 添加update_all_types参数会更新type_one和type_two中的text字段。

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/5.4/indices-put-mapping.html
