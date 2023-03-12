---
title: "ElasticSearch官网翻译_indexOptions"
date: 2017-05-19 16:24:00
tags: [Elasticsearch]
categories: [Search]
---

#### index_options（索引选项）

index_options参数控制什么信息添加到倒排索引，用于搜索和高亮显示。 它接受以下设置：

- docs : 只有文档编号被索引。 可以回答这个问题这个term词条是否存在于这个字段？

- freqs : 文件编号和term词条频率被索引。 term（词条）频率用于给搜索评分，重复出现的 term（词条）评分将高于单个出现的  term（词条）。

- positions : 文件编号，term词条频率和term词条位置（或顺序）被索引。 位置可以用于proximity or phrase queries（邻近或短语查询）。

- offsets : 文档编号，term词条频率，位置以及开始和结束字符偏移（将该term词条映射到原始字符串）被索引。 Offsets（偏移量）被用于postings highlighter。

分析的字符串字段使用positions（位置）作为默认值，所有其他字段使用docs作为默认值。

```
PUT my_index
{
  "mappings": {
    "my_type": {
      "properties": {
        "text": {
          "type": "string",
          "index_options": "offsets"
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

- (1) 默认情况下，text字段将使用postings highlighter，因为offsets（偏移量）已被索引。

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/2.4/index-options.html
