---
title: "ElasticSearch官网翻译_boost"
date: 2017-05-13 13:42:58
tags: [Elasticsearch]
categories: [Search]
---

#### boost（提升）

可以通过boost（提升）个别字段的权重 - 在索引时，更多地关注relevance score（相关性分数），boost参数如下：

```
PUT my_index
{
  "mappings": {
    "my_type": {
      "properties": {
        "title": {
          "type": "string",
          "boost": 2 
        },
        "content": {
          "type": "string"
        }
      }
    }
  }
}
```

- (1) 匹配到title字段的将是匹配到content字段的权重的2倍，其默认值为1.0。

请注意，title字段通常会比content字段短。 默认相关性计算考虑到字段长度，因此短的title字段将具有比长的content字段更高的自然boost（提升）。

###### 警告：为什么索引时使用boost（提升）是一个坏主意

我们建议不要在索引时使用boost（提升），原因如下：

<b>

- 你不能更改索引时的boost（提升）值，而不重新索引所有文档。
- 每个查询支持查询时设置boost（提升）值实现相同的效果。 不同之处在于，你可以调整boost（提升）值，而无需reindex。
- 索引时设置boost（提升）将存储为norm规范的一部分，只有一个字节。 这降低了字段长度归一化因子的解析度,从而降低相关性的计算的质量。

索引时设置boost（提升）的唯一优点是它被复制到_all字段中。 这意味着，当查询_all字段时，源自title字段的词条将具有比起源于content字段的词条更高的分数。 此功能的成本是：使用索引时设置boost（提升）值时_all字段上的查询变慢了。
</b>


参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/2.4/index-boost.html
