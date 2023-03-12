---
title: "ElasticSearch官网翻译_DynamicTemplates"
date: 2017-05-20 16:02:01
tags: [Elasticsearch]
categories: [Search]
---

#### Dynamic templates（动态模板）

<b>
动态模板允许你定义可应用于动态添加字段的自定义映射，基于如下：

- 由Elasticsearch使用match_mapping_type检测到的数据类型。
- 使用match和unmatch或者match_pattern规则的字段名称。
- 使用path_match和path_unmatch规则的全点路径的字段。

原始字段名称{name}和检测到的数据类型{dynamic_type}模板变量可以在映射规范中用作占位符。
</b>

###### 重要

<b>
仅当字段包含具体值（不为null空值或空数组）时才添加动态字段映射。 这意味着如果在dynamic_template中使用null_value选项，则只有在具有该字段的具体值已被索引的第一个文档之后才会应用该值。
</b>

动态模板被指定为一个命名对象的数组：

```
 "dynamic_templates": [
    {
      "my_template_name": { (1)
        ...  match conditions ... (2)
        "mapping": { ... } (3)
      }
    },
    ...
  ]
```
<b>

- (1) 模板名称可以是任何字符串值。
- (2) 匹配条件可以包括以下任何一种：match_mapping_type，match，match_pattern，unmatch，path_match，path_unmatch。
- (3) （上面匹配条件）匹配的字段应该使用的映射。

模板按顺序进行处理 - 第一个匹配模板即匹配成功。 可以使用PUT mapping API将新模板附加到列表的末尾。 如果新模板与现有模板具有相同的名称，它将替换旧版本。
</b>

##### match_mapping_type

<b>
match_mapping_type与通过动态字段映射检测到的数据类型匹配，换句话说，是Elasticsearch认为该字段应该具有的数据类型。 只能自动检测以下数据类型：boolean，date，double，long，object，string。 它也接受*匹配所有数据类型。
</b>

例如，如果我们要将所有integer字段映射为integer而不是long，并且所有string字段都analyzed和not_analyzed，我们可以使用以下模板：

```
PUT my_index
{
  "mappings": {
    "my_type": {
      "dynamic_templates": [
        {
          "integers": {
            "match_mapping_type": "long",
            "mapping": {
              "type": "integer"
            }
          }
        },
        {
          "strings": {
            "match_mapping_type": "string",
            "mapping": {
              "type": "string",
              "fields": {
                "raw": {
                  "type":  "string",
                  "index": "not_analyzed",
                  "ignore_above": 256
                }
              }
            }
          }
        }
      ]
    }
  }
}

PUT my_index/my_type/1
{
  "my_integer": 5, (1)
  "my_string": "Some string" (2)
}
```

- (1) my_integer字段被映射为integer类型。
- (2) my_string字段映射为analyzed分析字符串，并带有not_analyzed multi field（多字段）。 

##### match和unmatch

<b>
match参数使用模式匹配字段名称，而unmatch使用模式排除match匹配到的字段。

以下示例匹配名称以long_开头的所有字符串字段（除了以_text结尾的字符串除外），并将其映射为长字段：
</b>

```
PUT my_index
{
  "mappings": {
    "my_type": {
      "dynamic_templates": [
        {
          "longs_as_strings": {
            "match_mapping_type": "string",
            "match":   "long_*",
            "unmatch": "*_text",
            "mapping": {
              "type": "long"
            }
          }
        }
      ]
    }
  }
}

PUT my_index/my_type/1
{
  "long_num": "5", 
  "long_text": "foo" 
}
```

- (1) long_num字段被映射为long。
- (2) long_text字段使用默认字符串映射。

##### match_pattern

<b>
match_pattern参数调整match参数的行为，使其支持字段名称上的完整Java正则表达式匹配，而不是简单的通配符，例如：
</b>

```
  "match_pattern": "regex",
  "match": "^profit_\d+$"
```

##### path_match和path_unmatch

path_match和path_unmatch参数的工作方式与match和unmatch相同，但是执行在字段的完整点路径上，而不仅仅是最终名称。例如：some_object.*.some_field。

此示例将name对象中的任何字段的值复制到顶级full_name字段，但middle字段除外：

```
PUT my_index
{
  "mappings": {
    "my_type": {
      "dynamic_templates": [
        {
          "full_name": {
            "path_match":   "name.*",
            "path_unmatch": "*.middle",
            "mapping": {
              "type":       "string",
              "copy_to":    "full_name"
            }
          }
        }
      ]
    }
  }
}

PUT my_index/my_type/1
{
  "name": {
    "first":  "Alice",
    "middle": "Mary",
    "last":   "White"
  }
}
```

##### {name}和{dynamic_type}

{name}和{dynamic_type}占位符在映射中被替换为字段名称和检测到的动态类型。 以下示例将所有字符串字段设置为使用与该字段名称相同的analyzer分析器，并禁用所有非字符串字段的doc_values：

```
PUT my_index
{
  "mappings": {
    "my_type": {
      "dynamic_templates": [
        {
          "named_analyzers": {
            "match_mapping_type": "string",
            "match": "*",
            "mapping": {
              "type": "string",
              "analyzer": "{name}"
            }
          }
        },
        {
          "no_doc_values": {
            "match_mapping_type":"*",
            "mapping": {
              "type": "{dynamic_type}",
              "doc_values": false
            }
          }
        }
      ]
    }
  }
}

PUT my_index/my_type/1
{
  "english": "Some English text", (1)
  "count":   5 (2)
}
```

- (1) english字段被映射为一个string类型字段，并使用english analyzer分析器。
- (2) count字段被映射为一个long类型字段，并禁用doc_values

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/2.4/dynamic-templates.html
