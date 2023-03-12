---
title: "ElasticSearch官网翻译__idField"
date: 2017-05-12 22:55:06
tags: [Elasticsearch]
categories: [Search]
---

#### _id field

<b>
每个索引的文档都与一个_type相关联和一个_id。 _id字段不被索引，因为它的值可以从_uid字段自动获取。

_id字段的值可以在某些查询（term，terms，match，query_string，simple_query_string）中访问，但不能在聚合，脚本或排序中使用，而应使用_uid字段代替：
</b>

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
      "_id": [ "1", "2" ] (1)
    }
  }
}
```

- (1) 在_id字段上查询

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/2.4/mapping-id-field.html
