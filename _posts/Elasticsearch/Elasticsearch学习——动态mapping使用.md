---
title: "Elasticsearch学习——动态mapping使用"
date: 2017-02-12 17:42:02
tags: [Elasticsearch]
categories: [Search]
---

本文主要翻译自elastic.co的官方文档

### Dynamic Mapping

Elasticsearch要索引文档，不必首先创建索引，定义映射类型并定义字段。你可以直接创建索引文档，索引，类型和字段将自动生成。

```
PUT data/counters/1 
{ "count": 5 }
```

创建data索引，counters类型和名为count的字段，count字段的数据类型为long。

自动检测和添加新类型和字段称为dynamic mapping（动态映射）。 动态映射规则可以根据你的目的自定义：

- _default_ mapping（默认Mapping） : 配置要用于新映射类型的基本映射。
- Dynamic field mappings（动态字段Mapping） : 动态检测的规则。
- Dynamic templates（动态模板） : 自定义规则用于为动态添加的字段配置映射。

注意：Index Template允许你为新索引配置default mappings，settings，alias和warmer，无论是自动创建还是显式创建。

#### 禁止根据类型自动创建Mapping

通过将index.mapper.dynamic设置为false，可以禁用根据类型自动创建Mapping，可以通过在config / elasticsearch.yml文件中设置默认值，也可以将每个索引设置为索引设置：

```
PUT /_settings 
{
  "index.mapper.dynamic":false
}
```

- 禁用所有索引的根据类型自动创建Mapping。

无论此设置的值如何，仍然可以在创建索引或使用PUT映射API时显式添加Mapping。

### _default_ mapping

对于任何新映射类型，Default Mapping被用做基础的Mapping映射，可以通过向索引添加具有名称_default_的映射类型来定制，无论是在创建索引时还是在使用PUT映射API时。

```
PUT my_index
{
  "mappings": {
    "_default_": { 
      "_all": {
        "enabled": false
      }
    },
    "user": {}, 
    "blogpost": { 
      "_all": {
        "enabled": true
      }
    }
  }
}
```

- _default_ mapping将_all字段默认为disabled。
- user类型从_default_继承设置。
- blogpost类型覆盖默认值，并启用_all字段。


虽然_default_映射可以在创建索引之后更新，但是新默认值将仅影响之后创建的映射类型。

_default_映射可以与Index Template结合使用，以控制自动创建的索引中的动态创建的类型：

```
PUT _template/logging
{
  "template":   "logs-*", 
  "settings": { "number_of_shards": 1 }, 
  "mappings": {
    "_default_": {
      "_all": { 
        "enabled": false
      },
      "dynamic_templates": [
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

PUT logs-2015.10.01/event/1
{ "message": "error:16" }
```

- logging template将匹配任何以logs-开头的索引
- 将使用单个主分片创建匹配索引
- 对于新的根据类型创建的Mapping，默认情况下禁用_all字段
- String类型的字段将会被创建成analyzed的主字段，并且还有一个not_analyzed的.raw字段

### Dynamic templates

Dynamic templates允许你可以自定义Mapping并且应用于动态添加的字段，其基于：

- 由Elasticsearch检测的`datetype`，是否满足`match_mapping_type`。
- 该字段的名称，是否满足`match`和`un_match`或`match_pattern`。
- 该字段的完整的带.的路径，是否满足`path_match`和`path_unmatch`。

原始字段名{name}和检测到的数据类型{dynamic_type}模板变量可在Mapping格式中用作占位符。

注意：仅当字段包含具体值（不是空值或空数组）时，才会添加动态字段映射。这意味着如果在dynamic_template中使用了null_value选项，它将仅在第一个有具体值字段的文档之后应用。

Dynamic templates被指定为命名对象的数组：

```
"dynamic_templates": [
	{
	  "my_template_name": { 
	    ...  match conditions ... 
	    "mapping": { ... } 
	  }
	},
	...
]
```

- my_template_name : 模板名称可以是任何字符串值。
- match conditions : 匹配条件可以包括以下任何一个：`match_mapping_type`，`match`，`match_pattern`，`unmatch`，`path_match`，`path_unmatch`。
- mapping : 是设置匹配字段应使用的Mapping映射。

Template按顺序处理 - 第一个匹配模板即可。新的模板可以使用PUT映射API附加到列表的末尾。如果新模板与现有模板具有相同的名称，则它将替换旧版本。

#### match_mapping_type

match_mapping_type匹配由动态字段映射检测到的数据类型，换句话说，Elasticsearch认为字段应该具有的数据类型。只能自动检测以下数据类型：boolean，date，double，long，object，string。 它还接受*匹配所有数据类型。

例如，如果我们要将所有integer字段映射为integer而不是long，并将所有string字段同时analyzed和not_analyzed，则可以使用以下模板：

这里还需要重点看一下Dynamic field Mapping

- https://www.elastic.co/guide/en/elasticsearch/reference/2.3/dynamic-field-mapping.html

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
  "my_integer": 5, 
  "my_string": "Some string" 
}
```

- my_integer字段映射为integer。
- my_string字段映射为analyzed的字符串，和一个not_analyzed的.raw字段。

#### match和unmatch

match参数使用匹配字段名称的模式，而unmatch使用模式排除匹配的字段。

以下示例匹配名称以long_开头的所有字符串字段（除去了以_text结尾的字符串），并将它们映射为long字段：

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

- long_num字段映射为long类型。
- long_text字段使用default string mapping。

#### match_pattern

match_pattern参数调整match参数的行为，以便它支持对字段名称进行完全Java正则表达式匹配，而不是简单的通配符，例如：

```
  "match_pattern": "regex",
  "match": "^profit_\d+$"
```

#### path_match和path_unmatch

path_match和path_unmatch参数以与match和unmatch相同的方式工作，但在字段的完整的带.的路径上操作，而不仅仅是最终名称。some_object.*.some_field。

此示例将name对象中的任何字段的值复制到顶级full_name字段（middle字段除外）：

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

#### {name}和{dynamic_type}

在Mapping中{name}和{dynamic_type}占位符将替换为字段名称和检测到的动态类型。 以下示例将所有字符串字段设置为使用与字段名称相同的分析器，并对所有non-string字段禁用doc_values：

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
  "english": "Some English text", 
  "count":   5 
}
```

- english字段映射为english analyzer的string字段。
- count字段映射为禁用doc_values的long字段。

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/2.3/dynamic-mapping.html
- https://www.elastic.co/guide/en/elasticsearch/reference/2.3/default-mapping.html
- https://www.elastic.co/guide/en/elasticsearch/reference/2.3/dynamic-templates.html
- http://jiangpeng.info/blogs/2014/11/24/elasticsearch-mapping.html
- https://my.oschina.net/u/204498/blog/529955
- http://www.apache.wiki/pages/viewpage.action?pageId=9406922
- http://ericyy.me/2016/11/17/elasticsearch-02-mapping.html
- http://www.biglittleant.cn/2016/12/04/elastic-mapping/
- https://elasticsearch.cn/book/elasticsearch_definitive_guide_2.x/custom-dynamic-mapping.html
- http://aoyouzi.iteye.com/blog/2116599
- https://www.elastic.co/guide/en/elasticsearch/reference/2.4/query-dsl-nested-query.html
- https://github.com/elastic/kibana/issues/1084
- http://www.itkeyword.com/doc/7497179070907396685/nested-object-in-kibana-visualize
- https://tech.homeaway.com/development/2016/07/05/improving-kibana-query-language-with-nested-support.html