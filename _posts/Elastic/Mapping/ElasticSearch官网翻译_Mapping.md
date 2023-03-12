---
title: "ElasticSearch官网翻译_Mapping"
date: 2017-05-10 21:48:11
tags: [Elasticsearch]
categories: [Search]
---

#### Mapping（映射）

映射是定义如何存储和编制文档及其包含的字段的过程。举个例子，用映射来定义：

- 哪些字符串字段应被视为全文字段。
- 哪些字段包含数字，日期或地理位置。
- 文档中所有字段的值是否应该编入catch-all _all字段。
- 日期值的格式。
- 自定义规则来控制动态添加字段的映射。

#### Mapping Types（映射类型）

每个索引都有一个或多个映射类型，用于将索引中的文档划分为逻辑组。 用户文档可能存储在user类型中，并且博客文章以blogpost类型存储。

每个映射类型有：

- Meta-fields

Meta-fields用于定制文档的相关元数据。 Meta-fields的示例包括文档的_index，_type，_id和_source字段。

- 字段或属性

每个映射类型包含与该类型相关的字段或属性的列表。 user类型可能包含title，name和age字段，而blogpost类型可能包含title，body，user_id和created的字段。 相同索引中不同映射类型中具有相同名称的字段必须具有相同的映射。

##### 重要

在Elasticsearch版本2.0 - 2.3中，字段名称中不允许使用点。 Elasticsearch 2.4添加了允许点的设置，但是应谨慎使用此设置。

#### Field datatypes（字段数据类型）

每个字段都有一个数据类型，可以是：

- 一个简单的类型，如string，date，long，double，boolean或者ip。
- 一种支持JSON的层次性的类型，如对象或嵌套。
- 或者像geo_point，geo_shape，completion专门类型。

<b>为了不同的目的，以不同的方式索引相同的字段通常是有用的。 例如，字符串字段可以作为全文搜索的analyzed字段进行索引，并且作为用于排序或聚合的not_analyzed字段。 或者，你可以使用standard分析器，english和french分析器索引字符串字段。</b>

这是multi-fields的目的。 大多数数据类型通过fields参数支持multi-fields。

#### Dynamic mapping（动态映射）

字段和映射类型在使用之前不需要定义。 由于动态映射，新的映射类型和新的字段名称将自动添加，只需通过索引文档。 新的字段可以同时添加到顶级映射类型，也可以添加到内部object和nested字段。

动态映射规则可以配置为自定义的映射用于新类型和新字段。

#### Explicit mappings（显式映射）

你比Elasticsearch更了解你的数据，所以动态映射对于开始是有用的，在某些时候你会想要指定你自己的显式映射。

你可以在创建索引时创建映射类型和字段映射，并且可以使用PUT mapping API将映射类型和字段添加到现有索引。

#### 更新现有的mapping

<b>除了记录的文件外，现有的类型和字段映射不能更新。 更改映射将意味着已经编入索引的文档无效。 相反，你应该创建一个具有正确映射的新索引，并将你的数据重新索引到索引中。</b>

#### 字段在映射类型之间是共享的

映射类型用于对字段进行分组，但每个映射类型中的字段不是彼此独立的。 字段与：

- 同名
- 在同一个索引
- 在不同的映射类型
- 内部映射到同一个字段
- 并且必须具有相同的映射

<b>如果user和blogpost映射类型都存在title字段，则title字段必须在每种类型中具有完全相同的映射。 此规则的唯一例外是copy_to，dynamic，enabled，ignore_above，include_in_all和properties参数，每个字段可能有不同的设置。

通常，具有相同名称的字段也包含相同类型的数据，因此具有相同的映射不是问题。 当出现冲突时，可以通过选择更具描述性的名称来解决这些问题，比如user_title和blog_title。</b>

#### Example Mapping

可以在创建索引时指定上述示例的映射，如下所示：

```
PUT my_index (1)
{
  "mappings": {
    "user": { (2)
      "_all":       { "enabled": false  }, (3)
      "properties": { (4)
        "title":    { "type": "string"  }, (5)
        "name":     { "type": "string"  }, 
        "age":      { "type": "integer" }  
      }
    },
    "blogpost": { 
      "properties": { 
        "title":    { "type": "string"  }, 
        "body":     { "type": "string"  }, 
        "user_id":  {
          "type":   "string", 
          "index":  "not_analyzed"
        },
        "created":  {
          "type":   "date", 
          "format": "strict_date_optional_time||epoch_millis"
        }
      }
    }
  }
}
```
	
- (1) 创建一个名为my_index的索引
- (2) 添加映射类型user和blogpost
- (3) 禁用user映射类型的_all元字段
- (4) 指定每个映射类型中的字段或属性
- (5) 为每个字段指定数据类型和映射


参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/2.4/mapping.html
