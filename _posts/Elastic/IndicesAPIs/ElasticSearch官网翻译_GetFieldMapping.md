---
title: "ElasticSearch官网翻译_GetFieldMapping"
date: 2017-06-03 17:21:55
tags: [Elasticsearch]
categories: [Search]
---

## Get Field Mapping

Get field mappingAPI允许你取回一个或多个字段的映射定义。 当你不需要通过Get Mapping API返回的完整类型映射时，这很有用。

例如，考虑以下映射：

```
PUT publications
{
    "mappings": {
        "article": {
            "properties": {
                "id": { "type": "text" },
                "title":  { "type": "text"},
                "abstract": { "type": "text"},
                "author": {
                    "properties": {
                        "id": { "type": "text" },
                        "name": { "type": "text" }
                    }
                }
            }
        }
    }
}
```

以下内容仅返回title字段的映射：

```
GET publications/_mapping/article/field/title
```

响应是：

```
{
   "publications": {
      "mappings": {
         "article": {
            "title": {
               "full_name": "title",
               "mapping": {
                  "title": {
                     "type": "text"
                  }
               }
            }
         }
      }
   }
}
```

### Multiple Indices, Types and Fields（多索引，多类型，多字段）

Get field mapping API可用于通过单个调用从多个索引或类型获取多个字段的映射。 API的一般用法遵循以下语法：host:port/{index}/{type}/_mapping/field/{field}其中{index}，{type}和{field}可以代表逗号分隔的名称列表或通配符。 要获取所有索引的映射，你可以使用_all替换{index}。 以下是一些例子：

```
GET /twitter,kimchy/_mapping/field/message

GET /_all/_mapping/tweet,book/field/message,user.id

GET /_all/_mapping/tw*/field/*.id
```

### Specifying fields（指定字段）

Get mapping api允许你指定逗号分隔的字段列表。

例如，要选择author字段的id，你必须使用其全名author.id。

```
GET publications/_mapping/article/field/author.id,abstract,name
```

返回

```
{
   "publications": {
      "mappings": {
         "article": {
            "author.id": {
               "full_name": "author.id",
               "mapping": {
                  "id": {
                     "type": "text"
                  }
               }
            },
            "abstract": {
               "full_name": "abstract",
               "mapping": {
                  "abstract": {
                     "type": "text"
                  }
               }
            }
         }
      }
   }
}
```

Get field mapping API还支持通配符表示法。

```
GET publications/_mapping/article/field/a*
```

返回

```
{
   "publications": {
      "mappings": {
         "article": {
            "author.name": {
               "full_name": "author.name",
               "mapping": {
                  "name": {
                     "type": "text"
                  }
               }
            },
            "abstract": {
               "full_name": "abstract",
               "mapping": {
                  "abstract": {
                     "type": "text"
                  }
               }
            },
            "author.id": {
               "full_name": "author.id",
               "mapping": {
                  "id": {
                     "type": "text"
                  }
               }
            }
         }
      }
   }
}
```

### Other options（其他选项）

值|描述
---|---
include_defaults|对查询字符串添加include_defaults=true将导致响应包含默认值，这些默认值通常被抑制。

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/5.4/indices-get-field-mapping.html
