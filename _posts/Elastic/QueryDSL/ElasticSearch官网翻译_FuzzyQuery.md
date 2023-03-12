---
title: "ElasticSearch官网翻译_FuzzyQuery"
date: 2017-06-29 15:40:44
tags: [Elasticsearch]
categories: [Search]
---

## Fuzzy Query

模糊查询使用基于Levenshtein edit distance的相似度。

### String fields

模糊查询生成在fuzziness（模糊度）中指定的最大编辑距离之内的所有可能匹配的词条，然后检查词条字典，以找出哪些生成的词条在索引中实际存在。

这是一个简单的例子：

```
GET /_search
{
    "query": {
       "fuzzy" : { "user" : "ki" }
    }
}
```

或更高级的设置：

```
GET /_search
{
    "query": {
        "fuzzy" : {
            "user" : {
                "value" :         "ki",
                "boost" :         1.0,
                "fuzziness" :     2,
                "prefix_length" : 0,
                "max_expansions": 100
            }
        }
    }
}
```

#### Parameters

值|描述
---|---
fuzziness|最大编辑距离。 默认为AUTO。 请参阅"Fuzziness"一节。
prefix_length|不会"模糊化"的初始字符数。 这有助于减少必须检查的词条数量。 默认为0。
max_expansions|模糊查询将扩展到的最大词条数。 默认为50。

###### 警告

> 如果prefix_length设置为0，并且max_expansions设置为高数字，则该查询可能非常重。 这可能会导致索引中的每个词条都被检查！

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/5.4/query-dsl-fuzzy-query.html
