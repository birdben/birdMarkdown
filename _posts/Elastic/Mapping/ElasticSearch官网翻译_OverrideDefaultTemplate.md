---
title: "ElasticSearch官网翻译_OverrideDefaultTemplate"
date: 2017-05-20 17:26:27
tags: [Elasticsearch]
categories: [Search]
---

#### Override default template（覆盖默认模板）

你可以通过在匹配所有索引的索引模板中指定_default_类型映射，来覆盖所有索引和所有类型的默认映射。

例如，要为所有新索引中的所有类型默认禁用_all字段，你可以创建以下索引模板：

```
PUT _template/disable_all_field
{
  "disable_all_field": {
    "order": 0,
    "template": "*", (1)
    "mappings": {
      "_default_": { (2)
        "_all": { (3)
          "enabled": false
        }
      }
    }
  }
}
```

- (1) 将映射应用于与模式*匹配的索引，换句话说，将所有新的索引。
- (2) 定义索引中的_default_类型映射类型。
- (3) 默认情况下禁用_all字段。

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/2.4/override-default-template.html
