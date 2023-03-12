---
title: "ElasticSearch官网翻译_store"
date: 2017-05-20 14:01:03
tags: [Elasticsearch]
categories: [Search]
---

#### store（存储）

默认情况下，字段值被索引使其可搜索，但不stored（存储）。 这意味着该字段可以被查询，但不能原始字段值不能被retrieved（取回）。

通常这并不重要。 字段值已经是_source字段的一部分，它默认是存储的。 如果你只想retrieved（取回）单个字段或几个字段的值，而不是整个_source，则可以使用source filtering来实现。

<b>
在某些情况下，存储一个字段是有意义的。 例如，如果你有一个包含title，date和非常大的content字段的文档，则可能需要仅retrieved（取回）title和date，而无需从大的_source字段中提取这些字段：
</b>

```
PUT /my_index
{
  "mappings": {
    "my_type": {
      "properties": {
        "title": {
          "type": "string",
          "store": true 
        },
        "date": {
          "type": "date",
          "store": true 
        },
        "content": {
          "type": "string"
        }
      }
    }
  }
}

PUT /my_index/my_type/1
{
  "title":   "Some short title",
  "date":    "2015-01-01",
  "content": "A very long content field..."
}

GET my_index/_search
{
  "fields": [ "title", "date" ] 
}
```

- (1)(2) title和date字段被设置为存储。
- (3) 此请求将取回title和date字段的值。

###### 注意：存储的字段作为数组返回

<b>
为了一致性，存储的字段总是作为数组返回，因为无法知道原始字段值是单个值，多个值还是空数组。

如果你需要原始值，你应该从_source字段中取回它。

存储一个字段有意义的另一种情况是对于那些不出现在_source字段（例如copy_to字段）中的那些字段。
</b>

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/2.4/mapping-store.html
