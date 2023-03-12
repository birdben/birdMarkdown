---
title: "ElasticSearch官网翻译__uidField"
date: 2017-05-13 01:05:44
tags: [Elasticsearch]
categories: [Search]
---

#### _uid field

每个索引的文档都与一个_type相关联和一个_id。 这些值组合为{type}＃{id}并作为_uid字段编入索引。

_uid字段的值可以在查询，聚合，脚本以及排序时访问：

```
# Example documents
PUT my_index/my_type/1
{
  "text": "Document with ID 1"
}

PUT my_index/my_type/2
{
  "text": "Document with ID 2"
}

GET my_index/_search
{
  "query": {
    "terms": {
      "_uid": [ "my_type#1", "my_type#2" ] (1)
    }
  },
  "aggs": {
    "UIDs": {
      "terms": {
        "field": "_uid", (2)
        "size": 10
      }
    }
  },
  "sort": [
    {
      "_uid": { (3)
        "order": "desc"
      }
    }
  ],
  "script_fields": {
    "UID": {
      "script": "doc['_uid']" (4)
    }
  }
}
```

- (1) 在_uid字段上查询（也可以参见ids查询）
- (2) 在_uid字段上聚合
- (3) 在_uid字段上排序
- (4) 访问脚本中的_uid字段（必须启用inline scripts才能使此示例工作）


参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/2.4/mapping-uid-field.html
