---
title: "ElasticSearch官网翻译_DynamicFieldMapping"
date: 2017-05-20 15:30:39
tags: [Elasticsearch]
categories: [Search]
---

#### Dynamic field mapping（动态字段映射）

<b>
默认情况下，当在文档中找到以前未看到的字段时，Elasticsearch会将新字段添加到类型映射。 通过将dynamic（动态）参数设置为false或strict将其禁用，可以在文档和对象级别禁用此行为。
</b>

假设启用了dynamic field mapping（动态字段映射），一些简单的规则用于确定该字段应具有的数据类型：

JSON数据类型|Elasticsearch数据类型
---|---
null|没有字段会被添加
true或者false|boolean字段
浮点数|double字段
integer|long字段
object|object字段
array|取决于数组中第一个non-null的值
string|date日期字段（如果值通过日期检测），double字段或long字段（如果该值通过数字检测）或analyzed分析的字符串字段。

只有这些字段数据类型是可以被动态检测的。 所有其他数据类型必须明确映射。

除了以下列出的选项，动态字段映射规则也可以通过dynamic_templates进一步定制。

##### 日期检测

如果启用date_detection（默认），则会检查新的字符串字段，以查看其内容是否与dynamic_date_formats中指定的任何日期模式相匹配。 如果找到匹配，则添加一个新的日期字段和相应的格式。

dynamic_date_formats的默认值为：

[ "strict_date_optional_time","yyyy/MM/dd HH:mm:ss Z||yyyy/MM/dd Z"]

例如：

```
PUT my_index/my_type/1
{
  "create_date": "2015/09/02"
}

GET my_index/_mapping (1)
```

- (1) create_date字段已添加为格式为"yyyy/MM/dd HH:mm:ss Z||yyyy/MM/dd Z"的日期字段。

##### 禁用日期检测

可以通过将date_detection设置为false来禁用动态日期检测：

```
PUT my_index
{
  "mappings": {
    "my_type": {
      "date_detection": false
    }
  }
}

PUT my_index/my_type/1 
{
  "create": "2015/09/02"
}
```

- (1) create_date字段已添加为字符串字段。

##### 自定义检测到的日期格式

或者，可以自定义dynamic_date_formats以支持您自己的日期格式：

```
PUT my_index
{
  "mappings": {
    "my_type": {
      "dynamic_date_formats": ["MM/dd/yyyy"]
    }
  }
}

PUT my_index/my_type/1
{
  "create_date": "09/25/2015"
}
```

##### 数字检测

虽然JSON支持本机浮点数和整数数据类型，但有些应用程序或语言有时会将数字作为字符串处理。 通常正确的解决方案是显式映射这些字段，但可以启用数字检测（默认情况下禁用）可以自动执行此操作：

```
PUT my_index
{
  "mappings": {
    "my_type": {
      "numeric_detection": true
    }
  }
}

PUT my_index/my_type/1
{
  "my_float":   "1.0", 
  "my_integer": "1" 
}
```

- (1) my_float字段作为double字段添加。
- (2) my_integer字段作为long字段添加。

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/2.4/dynamic-field-mapping.html
