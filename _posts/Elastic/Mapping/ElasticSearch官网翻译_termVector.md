---
title: "ElasticSearch官网翻译_termVector"
date: 2017-05-20 14:20:51
tags: [Elasticsearch]
categories: [Search]
---

#### term_vector（词条向量）

Term vectors（词条向量）包含关于分析过程产生的terms（词条）的信息，包括：

- terms词条列表。
- 每个词条的位置（或顺序）。
- 映射词条的起始和结束字符与原始字符串原点的偏移量

可以存储这些term vectors（词条向量），以便可以以特定文档取回它们。

term_vector设置接受：

值|描述
---|---
no|没有term vectors（词条向量）被存储。 （默认）
yes|只有字段中的词条被存储。
with_positions|词条和位置都被存储
with_offsets|词条和字符偏移量被存储。
with_positions_offsets|词条，位置和字符偏移量被存储。

fast vector highlighter（快速向量高亮）需要with_positions_offsets。 term vectors API（词条向量API）可以取回任何存储的内容。

###### 警告

设置with_positions_offsets将使字段索引的大小增加一倍。

```
PUT my_index
{
  "mappings": {
    "my_type": {
      "properties": {
        "text": {
          "type":        "string",
          "term_vector": "with_positions_offsets"
        }
      }
    }
  }
}

PUT my_index/my_type/1
{
  "text": "Quick brown fox"
}

GET my_index/_search
{
  "query": {
    "match": {
      "text": "brown fox"
    }
  },
  "highlight": {
    "fields": {
      "text": {} (1)
    }
  }
}
```

- (1) 默认情况下，text字段将使用fast vector highlighter（快速向量高亮），因为term vectors（词条向量）被设置为启用。


参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/2.4/term-vector.html
