---
title: "ElasticSearch官网翻译_similarity"
date: 2017-05-20 13:45:07
tags: [Elasticsearch]
categories: [Search]
---

#### similarity（相似性）

Elasticsearch允许你配置每个字段的评分算法或相似度。 similarity（相似性设置）提供了一种简单方式来选择默认TF/IDF（例如BM25）以外的相似度算法。

相似性主要用于字符串字段，特别是analyzed分析的字符串字段，但也可以应用于其他字段类型。

可以通过调整内置相似度的参数来配置自定义相似度。 有关此高级选项的更多详细信息，请参阅similarity module（相似性模块）。

<b>
相似度设置可以开箱即用的，不需要任何配置是：

- default 

Elasticsearch和Lucene使用的默认TF/IDF算法。 有关更多信息，请参阅Lucene’s Practical Scoring Function（Lucene的实用评分功能）。

- BM25

Okapi BM25算法。 有关更多信息，请参阅Pluggable Similarity Algorithms（可插入相似度算法）。
</b>

当第一次创建一个字段时，可以在字段级别设置相似度，如下所示：

```
PUT my_index
{
  "mappings": {
    "my_type": {
      "properties": {
        "default_field": { (1)
          "type": "string"
        },
        "bm25_field": {
          "type": "string",
          "similarity": "BM25" (2)
        }
      }
    }
  }
}
```

- (1) default_field使用默认相似性（即TF/IDF）算法。
- (2) bm25_field使用BM25相似性算法。

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/2.4/similarity.html
