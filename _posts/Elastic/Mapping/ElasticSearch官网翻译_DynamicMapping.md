---
title: "ElasticSearch官网翻译_DynamicMapping"
date: 2017-05-20 14:53:53
tags: [Elasticsearch]
categories: [Search]
---

#### Dynamic Mapping（动态映射）

Elasticsearch最重要的功能之一是它尽可能快地开始探索数据。 要索引文档，你不必首先创建索引，定义映射类型和定义字段 - 你可以只索引文档，然后索引，类型和字段将自动创建：

```
PUT data/counters/1 (1)
{ "count": 5 }
```

- (1) 创建data索引，counters映射类型，以及一个名为count的字段，数据类型为long。

<b>
新自动检测新添加的类型和字段称为dynamic mapping（动态映射）。 可以根据你的目的定制动态映射规则：

- _default_ mapping : 配置的基础映射被用于新映射类型。
- Dynamic field mappings : 管理动态字段检测的规则。
- Dynamic templates : 配置动态添加字段映射的自定义规则。
</b>

###### 提示

<b>
Index Template（索引模板）允许你配置新索引的default mappings（默认映射），settings（设置），alias（别名）和warmers（加热器），无论是自动创建还是显式创建。
</b>

##### 禁用自动类型创建

通过在索引设置中将index.mapper.dynamic设置设置为false，可以禁用自动类型创建：

```
PUT /_settings (1)
{
  "index.mapper.dynamic":false
}
```

- (1) 禁用所有索引的自动类型创建。

无论此设置的值如何，仍可以通过creating an index（创建索引）或使用PUT mapping API显式添加类型。

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/2.4/dynamic-mapping.html
