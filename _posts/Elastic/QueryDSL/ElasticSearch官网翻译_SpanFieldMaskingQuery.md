---
title: "ElasticSearch官网翻译_SpanFieldMaskingQuery"
date: 2017-07-12 18:16:14
tags: [Elasticsearch]
categories: [Search]
---

## Span Field Masking Query

包装器允许span查询通过欺骗他们的搜索字段来参与复合单字段跨度查询。 span field masking query映射到Lucene的SpanFieldMaskingQuery

这可以用于支持跨越不同字段的span-near或span-or的查询，这通常不被允许。

当使用多个分析器索引相同的内容时，span field masking query与multi-fields多字段配合是非常有价值的。 例如，我们可以用标准分析器对一个字段进行索引，该分析器将文本分解成单词，再次用英语分析器将单词变成其词根形式。

例如：

```
GET /_search
{
  "query": {
    "span_near": {
      "clauses": [
        {
          "span_term": {
            "text": "quick brown"
          }
        },
        {
          "field_masking_span": {
            "query": {
              "span_term": {
                "text.stems": "fox"
              }
            },
            "field": "text"
          }
        }
      ],
      "slop": 5,
      "in_order": false
    }
  }
}
```

注意：作为span field masking query返回掩蔽字段，使用提供的字段名称的规范进行评分。 这可能导致意想不到的得分行为。

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/5.4/query-dsl-span-field-masking-query.html
