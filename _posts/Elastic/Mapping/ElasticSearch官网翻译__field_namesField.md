---
title: "ElasticSearch官网翻译__field_namesField"
date: 2017-05-12 22:18:04
tags: [Elasticsearch]
categories: [Search]
---

#### _field_names field

<b>
_field_names字段索引包含除null之外的任何值的文档中的每个字段的名称。 该字段使用exists和missing查询，以对特定字段查找有或者没有任何not-null（非空）值的文档。

_field_name字段的值可以在查询，聚合和脚本中访问：
</b>

```
# Example documents
PUT my_index/my_type/1
{
  "title": "This is a document"
}

PUT my_index/my_type/1
{
  "title": "This is another document",
  "body": "This document has a body"
}

GET my_index/_search
{
  "query": {
    "terms": {
      "_field_names": [ "title" ] 
    }
  },
  "aggs": {
    "Field names": {
      "terms": {
        "field": "_field_names", 
        "size": 10
      }
    }
  },
  "script_fields": {
    "Field names": {
      "script": "doc['_field_names']" 
    }
  }
}
```

- (1) 在_field_names字段上查询
- (2) 在_field_names字段上聚合
- (3) 访问脚本中的_field_names字段（必须启用inline scripts才能使此示例工作）


参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/2.4/mapping-field-names-field.html
