---
title: "ElasticSearch官网翻译_IndexAliases"
date: 2017-06-03 17:46:11
tags: [Elasticsearch]
categories: [Search]
---

## Index Aliases

若适用的情况下，Elasticsearch中的API接受一个索引名当针对特定的索引或几个索引。 Index aliases API允许给索引起个别名，所有API会自动将别名转换为实际的索引名称。 别名也可以映射到多个索引，当指定它时，别名将自动扩展到别名索引。 别名也可以与搜索时自动应用的过滤器和路由值相关联。 别名不能与索引具有相同的名称。

以下是将别名alias1与索引test1相关联的示例：

```
POST /_aliases
{
    "actions" : [
        { "add" : { "index" : "test1", "alias" : "alias1" } }
    ]
}
```

这里删除相同的别名：

```
POST /_aliases
{
    "actions" : [
        { "remove" : { "index" : "test1", "alias" : "alias1" } }
    ]
}
```

重命名别名是一个简单的remove，然后在同一API中add操作。 这个操作是原子的，无需担心别名不指向索引的很短的时间间隙：

```
POST /_aliases
{
    "actions" : [
        { "remove" : { "index" : "test1", "alias" : "alias1" } },
        { "add" : { "index" : "test2", "alias" : "alias1" } }
    ]
}
```

将别名与多个索引相关联只是几个add操作：

```
POST /_aliases
{
    "actions" : [
        { "add" : { "index" : "test1", "alias" : "alias1" } },
        { "add" : { "index" : "test2", "alias" : "alias1" } }
    ]
}
```

多个索引可以被指定使用indices数组语法的操作：

```
POST /_aliases
{
    "actions" : [
        { "add" : { "indices" : ["test1", "test2"], "alias" : "alias1" } }
    ]
}
```

要在一个操作中指定多个别名，也存在相应的aliases别名数组语法。

对于上面的示例，还可以使用glob模式将一个别名与多个共享一个通用名称的索引相关联：

```
POST /_aliases
{
    "actions" : [
        { "add" : { "index" : "test*", "alias" : "all_test_indices" } }
    ]
}
```

在这种情况下，别名是一个时间点时间别名，它将组合所有当前的匹配项，它将不会自动更新那些添加/删除与此模式匹配的新索引。

对指向多个索引的别名进行索引是一个错误。

也可以在一个操作中使用别名交换索引：

```
PUT test     
PUT test_2   
POST /_aliases
{
    "actions" : [
        { "add":  { "index": "test_2", "alias": "test" } },
        { "remove_index": { "index": "test" } }  
    ]
}
```

- (1) 我们错误添加的索引
- (2) 我们应该添加的索引
- (3) remove_index就像Delete Index一样

### Filtered Aliases（过滤的别名）

带有过滤器的别名提供了一种创建相同索引的不同“视图”的简单方法。 过滤器可以使用Query DSL定义，并使用此别名应用于所有Search，Count，Delete By Query和更多类似的操作。

要创建过滤的别名，首先我们需要确保映射中已经存在这些字段：

```
PUT /test1
{
  "mappings": {
    "type1": {
      "properties": {
        "user" : {
          "type": "keyword"
        }
      }
    }
  }
}
```

现在我们可以创建一个对user字段使用过滤器的别名：

```
POST /_aliases
{
    "actions" : [
        {
            "add" : {
                 "index" : "test1",
                 "alias" : "alias2",
                 "filter" : { "term" : { "user" : "kimchy" } }
            }
        }
    ]
}
```

#### Routing（路由）

可以将路由值与别名相关联。 此功能可以与过滤别名一起使用，以避免不必要的分片操作。

以下命令创建一个指向test索引的新别名别名alias1。 创建alias1之后，具有此别名的所有操作将自动修改为使用"1"进行路由：

```
POST /_aliases
{
    "actions" : [
        {
            "add" : {
                 "index" : "test",
                 "alias" : "alias1",
                 "routing" : "1"
            }
        }
    ]
}
```

还可以为搜索和索引操作指定不同的路由值：

```
POST /_aliases
{
    "actions" : [
        {
            "add" : {
                 "index" : "test",
                 "alias" : "alias2",
                 "search_routing" : "1,2",
                 "index_routing" : "2"
            }
        }
    ]
}
```

如上面的示例所示，搜索路由值可能包含以逗号分隔的多个值。 索引路由值可以只包含一个值。

如果使用路由别名的搜索操作也具有路由参数，则使用搜索别名路由值和参数中指定的路由值的交集。 例如，以下命令将使用"2"作为路由值：

```
GET /alias2/_search?q=user:kimchy&routing=2,3
```

如果使用索引路由别名的索引操作也具有parent routing父路由，则parent routing父路由将被忽略。

### Add a single alias（添加单一别名）

还可以使用endpoint添加别名

```
PUT /{index}/_alias/{name}
```

值|描述
---|---
index|别名所指的索引。 可以任何 * 或者 _all 或者 glob模式 或者 name1，name2，...
name|别名的名称。 这是一个必需的选项。
routing|可以与别名关联的可选路由。
filter|可以与别名关联的可选过滤器。

你也可以使用复数_aliases形式。

#### Examples:

- 添加基于时间的别名

```
PUT /logs_201305/_alias/2013
```

- 添加用户别名

首先创建索引并为user_id字段添加映射：

```
PUT /users
{
    "mappings" : {
        "user" : {
            "properties" : {
                "user_id" : {"type" : "integer"}
            }
        }
    }
}
```

然后添加特定用户的别名：

```
PUT /users/_alias/user_12
{
    "routing" : "12",
    "filter" : {
        "term" : {
            "user_id" : 12
        }
    }
}
```

### Aliases during index creation（索引时别名）

创建索引时也可以指定别名：

```
PUT /logs_20162801
{
    "mappings" : {
        "type" : {
            "properties" : {
                "year" : {"type" : "integer"}
            }
        }
    },
    "aliases" : {
        "current_day" : {},
        "2016" : {
            "filter" : {
                "term" : {"year" : 2016 }
            }
        }
    }
}
```

### Delete aliases（删除别名）

rest endpoint是: /{index}/_alias/{name}

值|描述
---|---
index|* 或者 _all 或者 glob pattern 或者 name1, name2, …
name|* 或者 _all 或者 glob pattern 或者 name1, name2, …

或者，你可以使用复数_aliases。 例：

```
DELETE /logs_20162801/_alias/current_day
```

### Retrieving existing aliases（获取现有别名）

Get index alias API允许通过别名和索引名称进行过滤。 此api重定向到master并获取请求的索引别名（如果可用）。 这个api只能连续找到的索引别名。

可能的选择：

值|描述
---|---
index|要获取别名的索引名称。 通过通配符支持部分名称，也可以使用逗号分隔多个索引名称。 还可以使用索引的别名。
alias|在响应中返回的别名的名称。 像索引选项一样，此选项支持通配符和选项，指定多个别名以逗号分隔。
ignore_unavailable|如果指定的索引名称不存在，该怎么办。如果设置为true，那么这些索引将被忽略。

rest endpoint是: /{index}/_alias/{alias}

#### Examples:（例子）

索引用户的所有别名：

```
GET /logs_20162801/_alias/*
```

响应：

```
{
 "logs_20162801" : {
   "aliases" : {
     "2016" : {
       "filter" : {
         "term" : {
           "year" : 2016
         }
       }
     }
   }
 }
}
```

任何索引中名为2016的所有别名：

```
GET /_alias/2016
```

响应：

```
{
  "logs_20162801" : {
    "aliases" : {
      "2016" : {
        "filter" : {
          "term" : {
            "year" : 2016
          }
        }
      }
    }
  }
}
```

在任何索引中以20开头的所有别名：

```
GET /_alias/20*
```

响应：

```
{
  "logs_20162801" : {
    "aliases" : {
      "2016" : {
        "filter" : {
          "term" : {
            "year" : 2016
          }
        }
      }
    }
  }
}
```

还有一个HEAD变体的get indices aliases api来检查索引别名是否存在。 索引aliases exists api支持与get indices aliases api相同的选项。 例子：

```
HEAD /_alias/2016
HEAD /_alias/20*
HEAD /logs_20162801/_alias/*
```

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/5.4/indices-aliases.html
