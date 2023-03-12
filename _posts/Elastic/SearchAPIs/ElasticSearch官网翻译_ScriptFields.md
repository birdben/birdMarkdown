---
title: "ElasticSearch官网翻译_ScriptFields"
date: 2017-05-24 21:51:45
tags: [Elasticsearch]
categories: [Search]
---

#### Script Fields

允许为每个命中返回script evaluation脚本评估（基于不同的字段），例如：

```
{
    "query" : {
        ...
    },
    "script_fields" : {
        "test1" : {
            "script" : "doc['my_field_name'].value * 2"
        },
        "test2" : {
            "script" : {
                "inline": "doc['my_field_name'].value * factor",
                "params" : {
                    "factor"  : 2.0
                }
            }
        }
    }
}
```

<b>
脚本字段可以在不存储的字段（上述情况下为my_field_name）中工作，并允许返回要返回的自定义值（脚本的评估值）。

脚本字段还可以访问索引的实际_source文档，并提取要从中返回的特定元素（可以是“对象”类型）。 </b>

这是一个例子：

```
    {
        "query" : {
            ...
        },
        "script_fields" : {
            "test1" : {
                "script" : "_source.obj1.obj2"
            }
        }
    }
```

注意_source关键字在这里导航类似json的模型。

<b>
理解doc['my_field'].value和_source.my_field之间的区别很重要。 第一，使用doc关键字将导致该字段的terms词条被加载到内存（缓存），这将导致更快的执行，但是更多的内存消耗。 此外，doc [...]符号只允许简单值的字段（不能从中返回一个json对象），只对非分析或单个词条的字段有意义。

另一方面，_source导致source源被加载，解析，然后只返回json的相关部分。
</b>

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/2.4/search-request-script-fields.html

