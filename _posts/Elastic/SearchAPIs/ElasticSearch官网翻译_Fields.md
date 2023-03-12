---
title: "ElasticSearch官网翻译_Fields"
date: 2017-05-24 21:28:56
tags: [Elasticsearch]
categories: [Search]
---

#### Fields

###### 警告

fields参数是关于明确标记为存储在映射中的字段，默认关闭，通常不推荐。 使用source filtering来选择要返回的原始源文档的子集。

允许选择性地加载搜索命中所表示的每个文档的特定存储字段。

```
{
    "fields" : ["user", "postDate"],
    "query" : {
        "term" : { "user" : "kimchy" }
    }
}
```

<b>fields还接受一个或多个通配符模式来控制应该返回文档的哪些字段。 警告：只能使用通配符模式检索stored存储的字段。</b>

例如：

```
{
    "fields": ["xxx*", "*xxx", "*xxx*", "xxx*yyy", "user", "postDate"],
    "query" : {
        "term" : { "user" : "kimchy" }
    }
}
```

*可用于从文档中加载所有存储的字段。

<b>一个空数组将只会返回每个命中的_id和_type，</b>例如：

```
{
    "fields" : [],
    "query" : {
        "term" : { "user" : "kimchy" }
    }
}
```

<b>
为了向后兼容，如果fields参数指定不存储的字段（store映射设置为false），它将加载_source并从中提取它。 此功能已被source filtering参数替换。

从文档中获取的字段值自身始终作为数组返回。 诸如_routing和_parent字段之类的元数据字段不会作为数组返回。

通过field选项也只能返回叶子字段。 因此，对象字段无法返回，并且这样的请求将失败。

脚本字段也可以自动检测并用作字段，因此，尽管不推荐使用_source.obj1.field1这样的东西，因为obj1.field1也可以正常使用。
</b>

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/2.4/search-request-fields.html

