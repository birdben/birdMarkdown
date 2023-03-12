---
title: "ElasticSearch官网翻译__parentField"
date: 2017-05-12 23:37:29
tags: [Elasticsearch]
categories: [Search]
---

#### _parent field

通过使一个映射类型成为另一个映射类型的父类，可以在同一索引中的文档之间建立parent-child（父子关系）：

```
PUT my_index
{
  "mappings": {
    "my_parent": {},
    "my_child": {
      "_parent": {
        "type": "my_parent" (1)
      }
    }
  }
}

PUT my_index/my_parent/1 (2)
{
  "text": "This is a parent document"
}

PUT my_index/my_child/2?parent=1 (3)
{
  "text": "This is a child document"
}

PUT my_index/my_child/3?parent=1 (4)
{
  "text": "This is another child document"
}

GET my_index/my_parent/_search
{
  "query": {
    "has_child": { (5)
      "type": "my_child",
      "query": {
        "match": {
          "text": "child document"
        }
      }
    }
  }
}
```

- (1) my_parent类型是my_child类型的父项。
- (2) 索引父文档。
- (3)(4) 索引两个子文档，指定父文档的ID。
- (5) 查找具有与查询匹配的子项的所有父文档。

有关更多信息，请参阅has_child和has_parent查询，children aggregation（子集合）和inner hits（内部命中）。

_parent字段的值可以在查询，聚合，脚本以及排序时访问：

```
GET my_index/_search
{
  "query": {
    "terms": {
      "_parent": [ "1" ] (1)
    }
  },
  "aggs": {
    "parents": {
      "terms": {
        "field": "_parent", (2)
        "size": 10
      }
    }
  },
  "sort": [
    {
      "_parent": { (3)
        "order": "desc"
      }
    }
  ],
  "script_fields": {
    "parent": {
      "script": "doc['_parent']" (4)
    }
  }
}
```

- (1) 在_parent字段上查询（另请参见has_parent查询和has_child查询）
- (2) 聚集在_parent字段（也可以看到child的聚合）
- (3) 在_parent字段上排序
- (4) 访问脚本中的_parent字段（必须启用inline scripts才能使此示例工作）

##### 父子限制

- 父类型和子类型必须不同 - 不能在相同类型的文档之间建立父子关系。
- _parent.type设置只能指向不存在的类型。 这意味着类型在创建之后不能成为父类型。
- <b>父文档和子文档必须在相同的分片上索引。 父ID用作子节点的路由值，以确保子节点与父节点在同一分片上进行索引。 这意味着在获取，删除或更新子文档时需要提供相同的父值。</b>

##### 全局序号

<b>
Parent-child使用全局序号来加快连接。 在对分片进行任何更改之后，需要重建全局序号。 分片中存储的父ID值越多，重建_parent字段的全局序号所需的时间越长。

默认情况下，全局序列是懒惰构建的：刷新后的第一个parent-child查询或聚合将触发构建全局序号。 这可能会为您的用户带来显着的延迟尖峰。 你可以使用eager_global_ordinals通过映射_parent字段将建立全局序号的成本从查询时间转移到刷新时间，如下所示：
</b>

```
PUT my_index
{
  "mappings": {
    "my_parent": {},
    "my_child": {
      "_parent": {
        "type": "my_parent",
        "fielddata": {
          "loading": "eager_global_ordinals"
        }
      }
    }
  }
}
```

可以检查全局序号使用的堆数量，如下所示：

```
# Per-index
GET _stats/fielddata?human&fields=_parent

# Per-node per-index
GET _nodes/stats/indices/fielddata?human&fields=_parent
```

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/2.4/mapping-parent-field.html
