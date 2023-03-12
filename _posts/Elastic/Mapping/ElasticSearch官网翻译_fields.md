---
title: "ElasticSearch官网翻译_fields"
date: 2017-05-19 20:31:42
tags: [Elasticsearch]
categories: [Search]
---

#### fields（字段）

为了不同的目的，以不同的方式索引相同的字段通常是有用的。 这是multi-fields的目的。 例如，string字段可以作为全文搜索的analyzed字段进行索引，并且作为用于排序或聚合的not_analyzed字段：

```
PUT /my_index
{
  "mappings": {
    "my_type": {
      "properties": {
        "city": {
          "type": "string",
          "fields": {
            "raw": { (1)
              "type":  "string",
              "index": "not_analyzed"
            }
          }
        }
      }
    }
  }
}

PUT /my_index/my_type/1
{
  "city": "New York"
}

PUT /my_index/my_type/2
{
  "city": "York"
}

GET /my_index/_search
{
  "query": {
    "match": {
      "city": "york" (2)
    }
  },
  "sort": {
    "city.raw": "asc" (3)
  },
  "aggs": {
    "Cities": {
      "terms": {
        "field": "city.raw" (4)
      }
    }
  }
}
```

- (1) city.raw字段是city字段的not_analyzed版本
- (2) analyzed的city字段可用于全文搜索
- (3)(4) city.raw字段可用于排序和聚合

###### 注意

多字段不会更改原始_source字段。

###### 提示

fields设置允许在同一索引中具有相同名称的字段的不同设置。 可以使用PUT mapping API将新的multi-fields（多字段）添加到现有字段中。

##### 具有多个分析器的多字段

multi-fields的另一个用例是以不同的方式分析相同的field以获得更好的相关性。 例如，我们可以使用standard analyzer（标准分析器）对一个字段进行索引，该分析器会将文本分解成单词，再次用english analyzer（英文分析器）将单词分解成其词根形式：

```
PUT my_index
{
  "mappings": {
    "my_type": {
      "properties": {
        "text": { (1)
          "type": "string",
          "fields": {
            "english": { (2)
              "type":     "string",
              "analyzer": "english"
            }
          }
        }
      }
    }
  }
}

PUT my_index/my_type/1
{ "text": "quick brown fox" } (3)

PUT my_index/my_type/2
{ "text": "quick brown foxes" } (4)

GET my_index/_search
{
  "query": {
    "multi_match": {
      "query": "quick brown foxes",
      "fields": [ (5)
        "text",
        "text.english"
      ],
      "type": "most_fields" (6)
    }
  }
}
```

- (1) text字段使用standard analyzer（标准分析器）。
- (2) text.english字段使用english analyzer（英语分析器）。
- (3)(4) 索引两个文档，一个是fox，另一个是foxes。
- <b>(5)(6) 查询text和text.english字段并组合分数。</b>

<b>
text字段在第一个文档中包含词条fox，第二个文档中词条foxes。 text.english字段在两个文档同时包含词词条fox，因为foxes是源于fox。

query string（查询字符串）为text字段使用standard analyzer（标准分析器）进行分析，为text.english字段使用english analyzer（英文分析器）进行分析。 衍生字段（text.english字段）允许对foxes查询也匹配只包含fox的文档。 这使我们能够尽可能多地匹配文档。 通过查询没有衍生的text字段，我们提高了匹配foxes的文档的相关性分数。
</b>

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/2.4/multi-fields.html

