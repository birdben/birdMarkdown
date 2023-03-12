---
title: "ElasticSearch官网翻译_includeInAll"
date: 2017-05-19 01:53:26
tags: [Elasticsearch]
categories: [Search]
---

#### include_in_all（_all包含字段）

include_in_all参数提供对每个字段控制是否包含在_all字段中。 它默认为true，除非index设置为no。

此示例演示如何从_all字段中排除date字段：

```
PUT my_index
{
  "mappings": {
    "my_type": {
      "properties": {
        "title": { (1)
          "type": "string"
        }
        "content": { (2)
          "type": "string"
        },
        "date": { (3)
          "type": "date",
          "include_in_all": false
        }
      }
    }
  }
}
```

- (1)(2) title和content字段将包含在_all字段中。
- (3) date字段不会包含在_all字段中。

###### 提示

include_in_all设置允许在相同索引中相同名称的字段有不同的设置。 可以使用PUT映射API在现有字段上更新其值。

<b>也可以在type级别和object或nested字段中设置include_in_all参数，在这种情况下，所有子字段都将继承该设置。 </b>例如：

```
PUT my_index
{
  "mappings": {
    "my_type": {
      "include_in_all": false, (1)
      "properties": {
        "title":          { "type": "string" },
        "author": {
          "include_in_all": true, (2)
          "properties": {
            "first_name": { "type": "string" },
            "last_name":  { "type": "string" }
          }
        },
        "editor": {
          "properties": {
            "first_name": { "type": "string" }, (3)
            "last_name":  { "type": "string", "include_in_all": true } (4)
          }
        }
      }
    }
  }
}
```

- (1) my_type中的所有字段都从_all中排除。
- (2) _all中包含author.first_name和author.last_name字段。
- (3)(4) 只有editor.last_name字段包含在_all中。 editor.first_name继承type级别的设置，并被排除。

###### 注意：Multi-fields和include_in_all

<b>
原始字段值被添加到_all字段，而不是字段分析器生成的term（词条）。 因此，由于每个Multi-fields（多字段）的值与其父字段完全相同，因此在Multi-fields（多字段）上将include_in_all设置为true是没有意义的。
</b>


参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/2.4/include-in-all.html
