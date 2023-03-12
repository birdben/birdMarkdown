---
title: "ElasticSearch官网翻译_SearchTemplate"
date: 2017-05-26 12:22:01
tags: [Elasticsearch]
categories: [Search]
---

#### Search Template

/_search/template endpoint允许在执行搜索请求之前使用mustache语言预呈现，并使用模板参数填充现有模板。

```
GET /_search/template
{
    "inline" : {
      "query": { "match" : { "{{my_field}}" : "{{my_value}}" } },
      "size" : "{{my_size}}"
    },
    "params" : {
        "my_field" : "foo",
        "my_value" : "bar",
        "my_size" : 5
    }
}
```

有关Mustache模板以及可以使用什么样的模板的更多信息，请查看online documentation of the mustache project。

Elasticsearch中实现的mustache语言是作为一种沙箱(sandboxed) 脚本语言，因此它遵守相关设置，这些设置可能是为了启用或禁用 脚本文档 中描述的每个语言、源和操作。

##### 更多模板示例

##### 使用单个值填充查询字符串

```
GET /_search/template
{
    "inline": {
        "query": {
            "match": {
                "title": "{{query_string}}"
            }
        }
    },
    "params": {
        "query_string": "search for these words"
    }
}
```

##### 传递一串数组

```
GET /_search/template
{
  "inline": {
    "query": {
      "terms": {
        "status": [
          "{{#status}}",
          "{{.}}",
          "{{/status}}"
        ]
      }
    }
  },
  "params": {
    "status": [ "pending", "published" ]
  }
}
```

其呈现为：

```
{
"query": {
  "terms": {
    "status": [ "", "pending", "", "published", "" ] (1)
  }
}
```

- (1) 只要""不是status字段的有效值，空的""将被忽略。

或者，你可以将status参数展开为以非转义（使用{{{}}}传递的字符串），如下所示：

```
GET /_search/template
{
  "inline": "{\"query\": {\"terms\": {\"status\": [ {{{status}}} ]}}}",
  "params": {
    "status": "\"pending\", \"published\""
  }
}
```

其呈现为：

```
{
  "query": {
    "terms": {
      "status": [
        "pending",
        "published"
      ]
    }
  }
}
```

##### 连接值的数组

可以使用{{#join}}数组{{/join}}函数将数组的值以逗号分隔的字符串连接起来：

```
GET /_search/template
{
  "inline": {
    "query": {
      "match": {
        "emails": "{{#join}}emails{{/join}}"
      }
    }
  },
  "params": {
    "emails": [ "username@email.com", "lastname@email.com" ]
  }
}
```

其呈现为：

```
{
    "query" : {
        "match" : {
            "emails" : "username@email.com,lastname@email.com"
        }
    }
}
```

该函数还接受一个自定义分隔符：

```
GET /_search/template
{
  "inline": {
    "query": {
      "range": {
        "born": {
            "gte"   : "{{date.min}}",
            "lte"   : "{{date.max}}",
            "format": "{{#join delimiter='||'}}date.formats{{/join delimiter='||'}}"
            }
      }
    }
  },
  "params": {
    "date": {
        "min": "2016",
        "max": "31/12/2017",
        "formats": ["dd/MM/yyyy", "yyyy"]
    }
  }
}
```

其呈现为：

```
{
    "query" : {
      "range" : {
        "born" : {
          "gte" : "2016",
          "lte" : "31/12/2017",
          "format" : "dd/MM/yyyy||yyyy"
        }
      }
    }
}
```

##### 默认值

例如，默认值为{{var}}{{^var}}default{{/var}}）

```
{
  "inline": {
    "query": {
      "range": {
        "line_no": {
          "gte": "{{start}}",
          "lte": "{{end}}{{^end}}20{{/end}}"
        }
      }
    }
  },
  "params": { ... }
}
```

当params是{"start":10, "end":15}时，这个查询将被呈现为：

```
{
    "range": {
        "line_no": {
            "gte": "10",
            "lte": "15"
        }
  }
}
```

但是当params是{"start":10}时，此查询将使用end的默认值：

```
{
    "range": {
        "line_no": {
            "gte": "10",
            "lte": "20"
        }
    }
}
```

##### 将参数转换为JSON

{{#tojson}}parameter{{/toJson}}函数可用于将maps和array等参数转换为JSON表示形式：

```
{
    "inline": "{\"query\":{\"bool\":{\"must\": {{#toJson}}clauses{{/toJson}} }}}",
    "params": {
        "clauses": [
            { "term": "foo" },
            { "term": "bar" }
        ]
   }
}
```

其呈现为：

```
{
    "query" : {
      "bool" : {
        "must" : [
          {
            "term" : "foo"
          },
          {
            "term" : "bar"
          }
        ]
      }
    }
}
```

##### 条件子句

不能使用模板的JSON格式来表达条件子句。 相反，模板必须作为字符串传递。 例如，假设我们想在line字段上运行match query，并且可选地要按行号进行过滤，其中start和end是可选的。

参数看起来像：

```
{
    "params": {
        "text":      "words to search for",
        "line_no": { (1)
            "start": 10, (2)
            "end":   20  (3)
        }
    }
}
```

- (1)(2)(3) 所有这三个元素都是可选的

我们可以将查询写成：

```
{
  "query": {
    "bool": {
      "must": {
        "match": {
          "line": "{{text}}" (1)
        }
      },
      "filter": {
        {{#line_no}} (2)
          "range": {
            "line_no": {
              {{#start}} (3)
                "gte": "{{start}}" (4)
                {{#end}},{{/end}} (5)
              {{/start}} (6)
              {{#end}} (7)
                "lte": "{{end}}" (8)
              {{/end}} (9)
            }
          }
        {{/line_no}} (10)
      }
    }
  }
}
```

- (1) 填写text参数的值
- (2)(10) 仅当指定了line_no时才包括range过滤器
- (3)(6) 仅当指定了line_no.start时才包含gte子句
- (4) 填写line_no.start参数的值
- (5) 仅当指定了line_no.start和line_no.end时，才在gte子句之后添加逗号
- (7)(9) 仅当指定了line_no.end时才包含lte子句
- (8) 填写line_no.end参数的值

###### 注意

如上所述，此模板无效JSON，因为它包含像{{#line_no}}这样的部分标记。 因此，模板应存储在文件中（参见“预先注册的templateedit”一节），或者通过REST API使用时，应将其写为字符串：

```
"inline": "{\"query\":{\"bool\":{\"must\":{\"match\":{\"line\":\"{{text}}\"}},\"filter\":{{{#line_no}}\"range\":{\"line_no\":{{{#start}}\"gte\":\"{{start}}\"{{#end}},{{/end}}{{/start}}{{#end}}\"lte\":\"{{end}}\"{{/end}}}}{{/line_no}}}}}}"
```

##### 预注册模板

你可以注册搜索模板并使用.mustache扩展名的文件将其存储在config/scripts目录中。 为了执行存储的模板，请在template key下引用它的名字：

```
GET /_search/template
{
    "file": "storedTemplate", (1)
    "params": {
        "query_string": "search for these words"
    }
}
```

- (1) config/scripts/中查询模板的名称，即：storedTemplate.mustache。

你还可以通过将名称为.scripts的特殊索引存储在Elasticsearch集群中来注册搜索模板。 有REST API来管理这些索引的模板。

```
POST /_search/template/<templatename>
{
    "template": {
        "query": {
            "match": {
                "title": "{{query_string}}"
            }
        }
    }
}
```

此模板可以被取回

```
GET /_search/template/<templatename>
```

其呈现为：

```
{
    "template": {
        "query": {
            "match": {
                "title": "{{query_string}}"
            }
        }
    }
}
```

此模板可以被删除

```
DELETE /_search/template/<templatename>
```

要在搜索时使用索引的模板，请使用：

```
GET /_search/template
{
    "id": "templateName", (1)
    "params": {
        "query_string": "search for these words"
    }
}
```

- (1) 存储在.scripts索引中的查询模板的名称。

##### 验证模板

可以使用给定参数在响应中呈现模板

```
GET /_render/template
{
  "inline": {
    "query": {
      "terms": {
        "status": [
          "{{#status}}",
          "{{.}}",
          "{{/status}}"
        ]
      }
    }
  },
  "params": {
    "status": [ "pending", "published" ]
  }
}
```

此调用将返回渲染的模板：

```
{
  "template_output": {
    "query": {
      "terms": {
        "status": [ (1)
          "pending",
          "published"
        ]
      }
    }
  }
}
```

- (1) status数组已使用params对象中的值进行填充。

文件和索引模板也可以通过分别替换文件inline或id的内联来呈现。 例如，渲染文件模板

```
GET /_render/template
{
  "file": "my_template",
  "params": {
    "status": [ "pending", "published" ]
  }
}
```

预注册的模板也可以使用

```
GET /_render/template/<template_name>
{
  "params": {
    "..."
  }
}
```

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/2.4/search-template.html

