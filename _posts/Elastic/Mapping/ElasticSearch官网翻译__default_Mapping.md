---
title: "ElasticSearch官网翻译__default_Mapping"
date: 2017-05-20 15:14:05
tags: [Elasticsearch]
categories: [Search]
---

#### _default_ mapping（默认映射）

<b>
默认映射，将用作任何新的映射类型的基本映射，在创建索引或以后使用PUT mapping API时，可以通过将名为_default_的映射类型添加到索引来进行定制。
</b>

```
PUT my_index
{
  "mappings": {
    "_default_": { (1)
      "_all": {
        "enabled": false
      }
    },
    "user": {}, (2)
    "blogpost": { (3)
      "_all": {
        "enabled": true
      }
    }
  }
}
```

<b>
- (1) _default_映射默认将_all字段禁用。
- (2) user类型从_default_继承设置。
- (3) blogpost类型覆盖默认值，并启用_all字段。

虽然_default_映射可以在创建索引后更新，但新的默认值仅会影响之后创建的映射类型。
</b>

<b>
_default_映射可以与Index Templates（索引模板）一起使用，以在自动创建的索引中控制动态创建的类型：
</b>

```
PUT _template/logging
{
  "template":   "logs-*", (1)
  "settings": { "number_of_shards": 1 }, (2)
  "mappings": {
    "_default_": {
      "_all": { (3)
        "enabled": false
      },
      "dynamic_templates": [
        {
          "strings": { (4)
            "match_mapping_type": "string",
            "mapping": {
              "type": "string",
              "fields": {
                "raw": {
                  "type":  "string",
                  "index": "not_analyzed",
                  "ignore_above": 256
                }
              }
            }
          }
        }
      ]
    }
  }
}

PUT logs-2015.10.01/event/1
{ "message": "error:16" }
```

<b>

- (1) logging模板将匹配以logs-开头的任何索引。
- (2) 将使用单个主分片创建匹配索引。
- (3) 默认情况下，新的类型映射中_all字段将被禁用。
- (4) 将使用analyzed分析的主字段和not_analyzed .raw字段创建字符串字段。
</b>

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/2.4/default-mapping.html
