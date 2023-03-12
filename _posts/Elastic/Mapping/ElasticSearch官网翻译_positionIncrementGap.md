---
title: "ElasticSearch官网翻译_positionIncrementGap"
date: 2017-05-20 11:24:59
tags: [Elasticsearch]
categories: [Search]
---

#### position_increment_gap（短语位置间隙）

<b>
Analyzed分析的字符串字段考虑了term（词条）位置，以便能够支持proximity or phrase queries（近似或短语查询）。 当使用多个值对字符串字段进行索引时，将在值之间添加“假”间隙，以防止大多数短语查询在整个值之间进行匹配。 此间隙的大小使用position_increment_gap配置，默认值为100。
</b>

例如：

```
PUT /my_index/groups/1
{
    "names": [ "John Abraham", "Lincoln Smith"]
}

GET /my_index/groups/_search
{
    "query": {
        "match_phrase": {
            "names": "Abraham Lincoln" (1)
        }
    }
}

GET /my_index/groups/_search
{
    "query": {
        "match_phrase": {
            "names": "Abraham Lincoln",
            "slop": 101 (2)
        }
    }
}
```

###### 个人觉得这里官方文档的第二个查询的语法有些问题，应该使用如下语句

```
GET /my_index/groups/_search
{
    "query": {
        "match_phrase": {
            "names": {
                "query": "John Abraham",
                "slop": 101 (2)
            }
        }
    }
}
```

- (1) <b>此短语查询与我们完全预期的文档不匹配。</b>
- (2) <b>这个短语查询与我们的文档相匹配，即使Abraham和Lincoln是分开的字符串，因为slop> position_increment_gap。</b>

position_increment_gap可以在映射中指定。 例如：

```
PUT my_index
{
  "mappings": {
    "my_type": {
      "properties": {
        "names": {
          "type": "string",
          "position_increment_gap": 0 
        }
      }
    }
  }
}

PUT /my_index/groups/1
{
    "names": [ "John Abraham", "Lincoln Smith"]
}

GET /my_index/groups/_search
{
    "query": {
        "match_phrase": {
            "names": "Abraham Lincoln" 
        }
    }
}
```

- (1) 下一个数组元素中的第一个term词条将与上一个数组元素中的最后一个term词条间隙为0。
- (2) 短语查询会很奇怪的匹配到我们的文档，但它是我们在映射中要求的。

###### 提示

position_increment_gap设置允许在同一索引中具有相同名称的字段的不同设置。 可以使用PUT mapping API在现有字段上更新其值。

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/2.4/position-increment-gap.html
