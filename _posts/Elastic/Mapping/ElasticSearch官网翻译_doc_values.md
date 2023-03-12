---
title: "ElasticSearch官网翻译_doc_values"
date: 2017-05-13 15:20:52
tags: [Elasticsearch]
categories: [Search]
---

#### doc_values（文档值）

<b>
默认情况下大多数字段都被索引，这使得它们可以搜索。inverted index（倒排索引）允许查询以唯一排序的term（词条）列表查找search term（搜索词），并且可以立即访问包含该term（词条）的文档列表。

排序，聚合和脚本中对字段值的访问需要不同的数据访问模式。我们不需要查找term（词条）和查找文档，而是可以查找文档并查找它在一个字段中的term（词条）。

Doc valus是在文档索引时构建的磁盘数据结构，这使得该数据访问模式成为可能。它们存储与_source相同的值，但是以一种以column-oriented（列）的方式存储，这样可以更有效地进行排序和聚合。几乎所有字段类型都支持Doc values，除了analyzed的字符串字段使用会抛出明显的异常。

默认情况下，支持doc values的所有字段都启用的。如果你确定不需要在字段上进行排序或聚合，或从脚本访问字段值，则可以禁用doc values以节省磁盘空间：
</b>

```
PUT my_index
{
  "mappings": {
    "my_type": {
      "properties": {
        "status_code": { (1)
          "type":       "string",
          "index":      "not_analyzed"
        },
        "session_id": { (2)
          "type":       "string",
          "index":      "not_analyzed",
          "doc_values": false
        }
      }
    }
  }
}
```

- (1) status_code字段默认启用了doc_values。
- (2) session_id已禁用doc_values，但仍可以查询。

###### 提示

doc_values设置允许在同一索引中具有相同名称的字段的不同设置。 可以使用PUT mapping API在现有字段上禁用（设置为false）。

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/2.4/doc-values.html
