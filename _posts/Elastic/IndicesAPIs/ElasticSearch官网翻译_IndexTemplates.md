---
title: "ElasticSearch官网翻译_IndexTemplates"
date: 2017-06-05 15:13:23
tags: [Elasticsearch]
categories: [Search]
---

## Index Templates

索引模板允许你定义将在创建新索引时自动应用的模板。 模板包括settings设置和mappings映射，以及一个简单模式的模板，用于控制模板是否应用于新的索引。

###### 注意

> 模板仅在索引创建时应用。 更改模板对现有索引没有影响。

举例：

```
PUT _template/template_1
{
  "template": "te*",
  "settings": {
    "number_of_shards": 1
  },
  "mappings": {
    "type1": {
      "_source": {
        "enabled": false
      },
      "properties": {
        "host_name": {
          "type": "keyword"
        },
        "created_at": {
          "type": "date",
          "format": "EEE MMM dd HH:mm:ss Z YYYY"
        }
      }
    }
  }
}
```

###### 注意

> 索引模板提供了C-style /* */ block注释。 在JSON文档中的任何地方都允许使用注释，除了在初始打开大括号之前。

定义名为template_1的模板，其模板模式为"te*"。 settings设置和mappings映射将应用于与"te*"模式匹配的任何索引名称。

还可以在索引模板中包含别名如下：

```
PUT _template/template_1
{
    "template" : "te*",
    "settings" : {
        "number_of_shards" : 1
    },
    "aliases" : {
        "alias1" : {},
        "alias2" : {
            "filter" : {
                "term" : {"user" : "kimchy" }
            },
            "routing" : "kimchy"
        },
        "{index}-alias" : {} 
    }
}
```

在索引创建过程中，别名中的{index}占位符将被替换为模板应用于的实际索引名称。

### Deleting a Template（删除模板）

索引模板通过名称（在上面的例子中为template_1）来标识，也可以用来被删除：

```
DELETE /_template/template_1
```

### Getting templates（获取索引模板）

索引模板通过名称（在上面的例子中是template_1）来标识，可以使用以下内容来取回：

```
GET /_template/template_1
```

你还可以使用通配符匹配多个模板，如：

```
GET /_template/temp*
GET /_template/template_1,template_2
```

要获取所有索引模板的列表，你可以运行：

```
GET /_template
```

### Template exists（模板是否存在）

用于检查模板是否存在。 例如：

```
HEAD _template/template_1
```

HTTP状态码表示是否存在给定名称的模板。 状态码200表示存在，而404表示不存在。

### Multiple Templates Matching（匹配多个模板）

多个索引模板可能会匹配同一个索引，在这种情况下，这两个settings设置和mappings映射将被合并到索引的最终配置中。 可以使用order参数来控制合并的顺序，首先应用较低的顺序，并且更高的次序覆盖它们。 例如：

```
PUT /_template/template_1
{
    "template" : "*",
    "order" : 0,
    "settings" : {
        "number_of_shards" : 1
    },
    "mappings" : {
        "type1" : {
            "_source" : { "enabled" : false }
        }
    }
}

PUT /_template/template_2
{
    "template" : "te*",
    "order" : 1,
    "settings" : {
        "number_of_shards" : 1
    },
    "mappings" : {
        "type1" : {
            "_source" : { "enabled" : true }
        }
    }
}
```

上述将禁用在所有type1类型上存储_source，但是对于以"te*"开头的索引，_source仍将被启用。 请注意，对于映射，合并是"deep"的，这意味着特定的基于对象/属性的映射可以轻松地在高阶模板上被添加/覆盖，低阶模板提供基础。

### Template Versioning（模板版本控制）

模板可以选择添加version版本号，可以是任何整数值，以便简化外部系统的模板管理。 version字段是完全可选的，仅用于模板的外部管理。 要取消设置version，只需替换模板没有指定verion即可。

```
PUT /_template/template_1
{
    "template" : "*",
    "order" : 0,
    "settings" : {
        "number_of_shards" : 1
    },
    "version": 123
}
```

要检查version字段，你可以使用filter_path过滤响应以限制只对该version的响应：

```
GET /_template/template_1?filter_path=*.version
```

这应该给出一个小的回应，使得它既容易又小成本地解析：

```
{
  "template_1" : {
    "version" : 123
  }
}
```

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/5.4/indices-templates.html
