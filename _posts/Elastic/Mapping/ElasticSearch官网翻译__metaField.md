---
title: "ElasticSearch官网翻译__metaField"
date: 2017-05-12 23:33:29
tags: [Elasticsearch]
categories: [Search]
---

#### _meta field

<b>
每个映射类型都可以有与之相关联的自定义元数据。 这些Elasticsearch完全不使用，但可用于存储应用程序特定的元数据，例如文档所属的类：
</b>

```
PUT my_index
{
  "mappings": {
    "user": {
      "_meta": { (1)
        "class": "MyApp::User",
        "version": {
          "min": "1.0",
          "max": "1.3"
        }
      }
    }
  }
}
```

- (1) 可以使用GET mapping API检索此_meta信息。

_meta字段可以使用PUT mapping API在现有类型上进行更新。

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/2.4/mapping-meta-field.html
