---
title: "ElasticSearch官网翻译__typeField"
date: 2017-05-13 01:00:06
tags: [Elasticsearch]
categories: [Search]
---

#### _type field

每个索引的文档都与一个_type相关联和一个_id。 为了使类型名称快速搜索，_type字段被索引。

_type字段的值可以在查询，聚合，脚本以及排序时访问：

```
# Example documents
PUT my_index/type_1/1
{
  "text": "Document with type 1"
}

PUT my_index/type_2/2
{
  "text": "Document with type 2"
}

GET my_index/_search/type_*
{
  "query": {
    "terms": {
      "_type": [ "type_1", "type_2" ] (1)
    }
  },
  "aggs": {
    "types": {
      "terms": {
        "field": "_type", (2)
        "size": 10
      }
    }
  },
  "sort": [
    {
      "_type": { (3)
        "order": "desc"
      }
    }
  ],
  "script_fields": {
    "type": {
      "script": "doc['_type']" (4)
    }
  }
}
```

- (1) 查询_type字段
- (2) 在_type字段上聚合
- (3) 在_type字段上排序
- (4) 访问脚本中的_type字段（必须启用inline scripts才能使此示例工作）

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/2.4/mapping-type-field.html
