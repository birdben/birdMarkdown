---
title: "ElasticSearch官网翻译__indexField"
date: 2017-05-12 22:55:06
tags: [Elasticsearch]
categories: [Search]
---

#### _index field

<b>
当跨多个索引执行查询时，有时需要添加仅与某些索引的文档相关联的查询子句。 _index字段允许在索引文档上进行索引的匹配。 它的价值在term，或terms查询，聚合，脚本以及排序时可以访问：

###### 注意
_index被显示为虚拟字段 - 它不会作为实际字段添加到Lucene索引。 这意味着你可以使用term或terms查询（或任何重写为term查询的查询，如match，query_string或simple_query_string查询）中的_index字段，但不支持prefix（前缀），wildcard（通配符），regexp（正则表达式）或fuzzy（模糊）查询。
</b>

```
# Example documents
PUT index_1/my_type/1
{
  "text": "Document in index 1"
}

PUT index_2/my_type/2
{
  "text": "Document in index 2"
}

GET index_1,index_2/_search
{
  "query": {
    "terms": {
      "_index": ["index_1", "index_2"] (1)
    }
  },
  "aggs": {
    "indices": {
      "terms": {
        "field": "_index", (2)
        "size": 10
      }
    }
  },
  "sort": [
    {
      "_index": { (3)
        "order": "asc"
      }
    }
  ],
  "script_fields": {
    "index_name": {
      "script": "doc['_index']" (4)
    }
  }
}
```

- (1) 查询_index字段
- (2) 聚集在_index字段上
- (3) 在_index字段上排序
- (4) 访问脚本中的_index字段（必须启用inline scripts才能使此示例正常工作）

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/2.4/mapping-index-field.html
