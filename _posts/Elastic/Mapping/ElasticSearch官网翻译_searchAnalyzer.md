---
title: "ElasticSearch官网翻译_searchAnalyzer"
date: 2017-05-20 13:30:31
tags: [Elasticsearch]
categories: [Search]
---

#### search_analyzer（搜索分析器）

<b>
通常，在索引时和搜索时应该应用相同的分析器，以确保查询中的terms（词条）与倒排索引中的terms（词条）格式相同。

有时候，在搜索时使用不同的分析器，例如在使用edge_ngram tokenizer进行自动填充时，这是有意义的。

默认情况下，查询将使用字段映射中定义的analyzer分析器，但可以使用search_analyzer设置来覆盖这一点：
</b>

```
PUT /my_index
{
  "settings": {
    "analysis": {
      "filter": {
        "autocomplete_filter": {
          "type": "edge_ngram",
          "min_gram": 1,
          "max_gram": 20
        }
      },
      "analyzer": {
        "autocomplete": { (1)
          "type": "custom",
          "tokenizer": "standard",
          "filter": [
            "lowercase",
            "autocomplete_filter"
          ]
        }
      }
    }
  },
  "mappings": {
    "my_type": {
      "properties": {
        "text": {
          "type": "string",
          "analyzer": "autocomplete", (2)
          "search_analyzer": "standard" (3)
        }
      }
    }
  }
}

PUT my_index/my_type/1
{
  "text": "Quick Brown Fox" (4)
}

GET my_index/_search
{
  "query": {
    "match": {
      "text": {
        "query": "Quick Br", (5)
        "operator": "and"
      }
    }
  }
}
```

- (1) 分析器设置使用自定义autocomplete（自动填充）分析器。
- (2)(3) text字段在索引时使用autocomplete（自动填充）分析器，但在搜索时使用standard（标准）分析器。
- (4) 这个字段被索引为如下词条：[q，qu，qui，quic，quick，b，br，bro，brow，brown，f，fo，fox]
- (5) 该查询搜索这两个词条：[quick，br]

有关此示例的完整说明，请参阅Index time search-as-you-type。

###### 提示

<b>
在同一个索引中，相同名称的字段的search_analyzer设置必须相同。 可以使用PUT mapping API在现有字段上更新其值。
</b>

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/2.4/search-analyzer.html
