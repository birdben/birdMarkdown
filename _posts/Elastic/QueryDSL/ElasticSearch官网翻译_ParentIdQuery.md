---
title: "ElasticSearch官网翻译_ParentIdQuery"
date: 2017-06-30 17:44:08
tags: [Elasticsearch]
categories: [Search]
---

## Parent Id Query

###### 注意

> 在5.0.0中添加

parent_id查询可用于查找属于特定父项的子文档。 给定以下映射定义：

```
PUT /my_index
{
    "mappings": {
        "blog_post": {
            "properties": {
                "name": {
                    "type": "keyword"
                }
            }
        },
        "blog_tag": {
            "_parent": {
                "type": "blog_post"
            },
            "_routing": {
                "required": true
            }
        }
    }
}
```

```
GET /my_index/_search
{
    "query": {
        "parent_id" : {
            "type" : "blog_tag",
            "id" : "1"
        }
    }
}
```

上面的功能相当于使用以下has_parent查询，但执行得更好，因为它不需要进行连接：

```
GET /my_index/_search
{
  "query": {
    "has_parent": {
      "parent_type": "blog_post",
        "query": {
          "term": {
            "_id": "1"
        }
      }
    }
  }
}
```

#### Parameters

此查询具有两个必需参数：

值|描述
---|---
type|child类型。 这必须是一个_parent字段定义的类型。
id|查询文档必须参照指定的parent文档的id。
ignore_unmapped|当设置为true时，将忽略未映射的type，并且将不匹配此查询的任何文档。 当对可能具有不同映射的多个索引进行查询时，这可能很有用。 当设置为false（默认值）时，如果type未映射，则查询将抛出异常。

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/5.4/query-dsl-parent-id-query.html
