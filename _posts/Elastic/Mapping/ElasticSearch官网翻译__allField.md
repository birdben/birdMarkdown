---
title: "ElasticSearch官网翻译__allField"
date: 2017-05-12 14:12:50
tags: [Elasticsearch]
categories: [Search]
---

#### _all field

_all字段是一个特殊的catch-all字段，它将所有其他字段的值连接成一个大字符串，使用空格作为分隔符，然后对其进行analyzed（分析）和indexed（索引），但不stored（存储）。 这意味着它可以被searched（搜索），但不能被retrieved（检索）。

_all字段允许你搜索文档中的值，而不知道哪个字段包含该值。 这使得它成为开始使用新dataset（数据集）时的有用选项。 例如：

```
PUT my_index/user/1 (1)
{
  "first_name":    "John",
  "last_name":     "Smith",
  "date_of_birth": "1970-10-24"
}

GET my_index/_search
{
  "query": {
    "match": {
      "_all": "john smith 1970"
    }
  }
}
```

- (1) _all字段将包含以下terms（词条）：[ "john", "smith", "1970", "10", "24" ]

###### 注意：所有值被视为字符串

<b>
上述示例中的date_of_birth字段被识别为日期字段，因此将索引表示1970-10-24 00:00:00 UTC的单个term（词条）。 然而，_all字段将所有值视为字符串，因此日期值被索引为三个字符串terms（词条）："1970", "24", "10"。

重要的是注意，_all字段将来自每个字段的原始值作为字符串组合。 它不组合来自每个字段的terms（词条）。
</b>

_all字段只是一个字符串字段，并接受与其他字符串字段相同的参数，包括analyzer，term_vectors，index_options和store。

<b>
_all字段可以是有用的，特别是当使用简单的过滤来探索新的数据时。 但是，通过将字段值连接成一个大字符串，_all字段将丢失短字段（更相关）和长字段（较不相关）之间的区别。 对于搜索相关性很重要的用例，最好专门查询各个字段。
</b>

_all字段不是没有成本的：它需要额外的CPU周期并使用更多的磁盘空间。 如果不需要使用，它可以完全禁用或per-field basis（按照字段定制）。

##### 在查询中使用_all字段

<b>query_string和simple_query_string查询默认查询_all字段，除非指定了另一个字段：</b>

```
GET _search
{
  "query": {
    "query_string": {
      "query": "john smith 1970"
    }
  }
}
```

在URI搜索请求中的?q=参数也同样可以写成如下（内部改写为query_string查询）：

```
GET _search?q=john+smith+1970
```

其他查询（如match匹配和term精确查询）要求你按照第一个示例显式指定_all字段。

##### 禁用_all字段

通过将enabled设置为false，可以对_all字段完全禁用：

```
PUT my_index
{
  "mappings": {
    "type_1": { (1)
      "properties": {...}
    },
    "type_2": { (2)
      "_all": {
        "enabled": false
      },
      "properties": {...}
    }
  }
}
```

- (1) type_1中的_all字段已启用。
- (2) type_2中的_all字段已完全禁用。

<b>如果_all字段被禁用，则URI搜索请求和query_string和simple_query_string查询将无法将其用于查询。 你可以将它们配置为使用与index.query.default_field设置不同的字段：</b>

```
PUT my_index
{
  "mappings": {
    "my_type": {
      "_all": {
        "enabled": false (1)
      },
      "properties": {
        "content": {
          "type": "string"
        }
      }
    }
  },
  "settings": {
    "index.query.default_field": "content" (2)
  },
}
```

- (1) _all字段对于my_type类型是禁用的。
- (2) query_string查询将默认查询此索引中的content字段。

##### 排除_all的字段

<b>可以使用include_in_all设置从_all字段中包含或排除各个字段。</b>

##### 索引提升和_all字段

<b>使用boost参数，索引时可以提升各个字段的权重。 _all字段考虑到这些权重提升：</b>

```
PUT myindex
{
  "mappings": {
    "mytype": {
      "properties": {
        "title": { 
          "type": "string",
          "boost": 2
        },
        "content": { 
          "type": "string"
        }
      }
    }
  }
}
```

- <b>(1)(2) 当查询_all字段时，源于title字段的字符的相关性是源于content字段的字符的相关性的两倍。</b>

###### 警告

使用 _all 字段的在索引时 boost 对查询性能有显着的影响。 通常更好的解决方案是单独查询字段，可在查询时 boost。

##### 自定义_all字段

虽然每个索引只有一个_all字段，但copy_to参数允许创建多个自定义_all字段。 例如，first_name和last_name字段可以组合到full_name字段中：

```
PUT myindex
{
  "mappings": {
    "mytype": {
      "properties": {
        "first_name": {
          "type":    "string",
          "copy_to": "full_name" (1)
        },
        "last_name": {
          "type":    "string",
          "copy_to": "full_name" (2)
        },
        "full_name": {
          "type":    "string"
        }
      }
    }
  }
}

PUT myindex/mytype/1
{
  "first_name": "John",
  "last_name": "Smith"
}

GET myindex/_search
{
  "query": {
    "match": {
      "full_name": "John Smith"
    }
  }
}
```

- (1)(2) first_name和last_name值将复制到full_name字段。

##### 高亮显示和_all字段

<b>
一个字段只能用于高亮显示如果原始字符串值可用，无论是从_source字段或store字段可用。

_all字段不在_source字段中，默认情况下不存储，因此无法高亮显示。 有两个选项。 存储_all字段或高亮显示原始字段。
</b>

##### 存储_all字段

如果store设置为true，则可以retrievable（检索）原始字段值，并且可以高亮显示：

```
PUT myindex
{
  "mappings": {
    "mytype": {
      "_all": {
        "store": true
      }
    }
  }
}

PUT myindex/mytype/1
{
  "first_name": "John",
  "last_name": "Smith"
}

GET _search
{
  "query": {
    "match": {
      "_all": "John Smith"
    }
  },
  "highlight": {
    "fields": {
      "_all": {}
    }
  }
}
```

当然，存储_all字段将使用明显更多的磁盘空间，并且由于它是其他字段的组合，所以可能会导致奇怪的高亮显示结果。

_all字段也接受term_vector和index_options参数，允许使用the fast vector highlighter（快速矢量高亮）和postings highlighter（帖子高亮）。

##### 高亮显示原始字段

<b>你可以查询_all字段，但使用原始字段进行高亮显示，如下所示：</b>

```
PUT myindex
{
  "mappings": {
    "mytype": {
      "_all": {}
    }
  }
}

PUT myindex/mytype/1
{
  "first_name": "John",
  "last_name": "Smith"
}

GET _search
{
  "query": {
    "match": {
      "_all": "John Smith" (1)
    }
  },
  "highlight": {
    "fields": {
      "*_name": { (2)
        "require_field_match": "false"  (3)
      }
    }
  }
}
```

- (1) <b>查询检查_all字段以查找匹配的文档。</b>
- (2) <b>在_source可以使用的两个name字段上执行突出显示。</b>
- (3) <b>该查询未针对name字段运行，因此将require_field_match设置为false。</b>


参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/2.4/mapping-all-field.html
