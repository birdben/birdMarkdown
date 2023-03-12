---
title: "ElasticSearch官网翻译_norms"
date: 2017-05-20 10:59:34
tags: [Elasticsearch]
categories: [Search]
---

#### norms（标准信息）

Norms存储各种归一化因子 - 一个数字来表示相对字段长度和index time boost（索引时提升权重）设置，这些数据稍后在查询时使用，以便相对于查询计算文档的分数。

虽然对评分有用，但norms也需要相当大的内存（通常是为了索引每个文档中每个字段每个字节的顺序，甚至不包含此字段的文档也是如此）。 因此，如果你不需要在特定字段上计算得分，则应禁用该字段的norms。 特别地，仅用于过滤或聚合的字段就是这种情况。

###### 提示

对同一个索引中同名的字段，norms.enabled设置必须具有相同的设置。 可以使用PUT mapping API在现有字段上禁用norms。

实际上norms可以被禁用（但不能重新启用），如下所示使用PUT mapping API：

```
PUT my_index/_mapping/my_type
{
  "properties": {
    "title": {
      "type": "string",
      "norms": {
        "enabled": false
      }
    }
  }
}
```

###### 注意

<b>
Norms不会立即被删除，但是当你继续索引新文档时，旧段将合并到新的段中，将被删除。 对已经删除norms的字段进行任何得分计算可能会返回不一致的结果，因为某些文档不再具有norms，而其他文档可能仍然有norms。
</b>

##### 延迟加载norms

<b>
norms可以即时地（eager）加载到内存中，每当新的分段上线时，或者只有当查询该字段时，它们才能被延迟加载（lazy，default）。
</b>

即时加载可以按如下方式配置：

```
PUT my_index/_mapping/my_type
{
  "properties": {
    "title": {
      "type": "string",
      "norms": {
        "loading": "eager"
      }
    }
  }
}
```

###### 提示

在同一索引中，norms.loading设置必须具有相同名称的字段相同的设置。 可以使用PUT mapping API在现有字段上更新其值。


参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/2.4/norms.html

