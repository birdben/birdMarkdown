---
title: "ElasticSearch官网翻译__sourceField"
date: 2017-05-13 00:38:02
tags: [Elasticsearch]
categories: [Search]
---

#### _source field

<b>
_source字段包含在索引时传递的原始JSON文档正文。 _source字段本身不被索引（因此是不可搜索的），但它被存储，以便在执行fetch请求时可以返回，例如get或search。
</b>

##### 禁用_source字段

虽然很方便，_source字段确实在索引中引起存储开销。 因此，可以禁用如下：

```
PUT tweets
{
  "mappings": {
    "tweet": {
      "_source": {
        "enabled": false
      }
    }
  }
}
```

###### 在禁用_source字段之前请考虑

用户经常禁用_source字段而不考虑后果，然后后悔。 如果_source字段不可用，则不支持许多功能：

- update API。
- 在高亮显示。
- 将索引从一个Elasticsearch索引重定向到另一个索引的能力，以更改映射或分析，或将索引升级到新的主要版本。
- 通过查看索引时使用的原始文档来调试查询或聚合的能力。
- 潜在的未来可能会自动修复索引损坏的能力。

###### 提示

如果磁盘空间是一个问题，而是增加compression level（压缩级别）而不是禁用_source。

##### 指标用例

<b>
指标用例与其他基于时间或日志记录的用例不同，因为有许多小型文档只包含数字，日期或关键字。 没有更新，没有突出显示的请求，并且数据快速老化，因此不需要reindex。 搜索请求通常使用简单查询来按日期或标记过滤数据集，并将结果作为聚合返回。

在这种情况下，禁用_source字段将节省空间并减少I / O。 建议在指标情况下禁用_all字段。
</b>

##### 包含/排除_source中的字段

专家功能是在文档索引后但在_source字段存储之前修剪_source字段的内容的能力。

###### 警告

从_source中删除字段具有类似于禁用_source的缺点，特别是你无法将一个Elasticsearch索引的索引文档重新索引到另一个。 考虑使用source filtering。

可以使用include/excludes参数（也可以接受通配符），如下所示：

```
PUT logs
{
  "mappings": {
    "event": {
      "_source": {
        "includes": [
          "*.count",
          "meta.*"
        ],
        "excludes": [
          "meta.description",
          "meta.other.*"
        ]
      }
    }
  }
}

PUT logs/event/1
{
  "requests": {
    "count": 10,
    "foo": "bar" 
  },
  "meta": {
    "name": "Some metric",
    "description": "Some metric description", 
    "other": {
      "foo": "one", 
      "baz": "two" 
    }
  }
}

GET logs/event/_search
{
  "query": {
    "match": {
      "meta.other.foo": "one" 
    }
  }
}
```

- <b>(1)(2)(3)(4) 这些字段将从存储的_source字段中删除。</b>
- <b>(5) 我们仍然可以搜索这个字段，即使它不存储在_source。</b>

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/2.4/mapping-source-field.html
