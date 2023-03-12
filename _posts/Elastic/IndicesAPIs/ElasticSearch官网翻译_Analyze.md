---
title: "ElasticSearch官网翻译_Analyze"
date: 2017-06-05 14:36:29
tags: [Elasticsearch]
categories: [Search]
---

## Analyze

对文本执行分析过程，并返回文本分解的tokens（标记）。

可以在不指定索引的情况下使用，用于许多内置分析器之一：

```
GET _analyze
{
  "analyzer" : "standard",
  "text" : "this is a test"
}
```

如果text参数作为字符串数组提供，则将其分析为multi-valued多值字段。

```
GET _analyze
{
  "analyzer" : "standard",
  "text" : ["this is a test", "the second text"]
}
```

或者通过从tokenizers（标记生成器），token filters（标记过滤器）和char filters（字符过滤器）构建定制的瞬态分析器。 标记过滤器可以使用较短的过滤器参数名称：

```
GET _analyze
{
  "tokenizer" : "keyword",
  "filter" : ["lowercase"],
  "text" : "this is a test"
}
```

```
GET _analyze
{
  "tokenizer" : "keyword",
  "filter" : ["lowercase"],
  "char_filter" : ["html_strip"],
  "text" : "this is a <b>test</b>"
}
```

###### 警告：Deprecated in 5.0.0.

> 使用filter/char_filter替换已被删除的filters/char_filters和token_filters

可以在请求正文中指定自定义tokenizers（标记生成器），token filters（标记过滤器）和char filters（字符过滤器），如下所示：

```
GET _analyze
{
  "tokenizer" : "whitespace",
  "filter" : ["lowercase", {"type": "stop", "stopwords": ["a", "is", "this"]}],
  "text" : "this is a test"
}
```

它也可以针对特定索引运行：

```
GET twitter/_analyze
{
  "text" : "this is a test"
}
```

以上将对"this is a test"文本进行分析，使用与twitter索引相关联的默认索引分析器。 也可以使用analyzer分析器来使用不同的分析器：

```
GET twitter/_analyze
{
  "analyzer" : "whitespace",
  "text" : "this is a test"
}
```

此外，可以从基于字段映射中获得分析器，例如：

```
GET twitter/_analyze
{
  "field" : "obj1.field1",
  "text" : "this is a test"
}
```

将导致基于在obj1.field1映射中配置的分析器进行分析（如果obj1.field1没有配置分析器，将使用默认索引分析器）。

###### 警告

> 在5.1.0中已被弃用，请求参数已经不推荐使用，并将在下一个主要版本中删除。 请使用JSON参数，而不是请求参数。

所有参数也可以作为请求参数提供。 例如：

```
GET /_analyze?tokenizer=keyword&filter=lowercase&text=this+is+a+test
```

为了向后兼容，我们也接受text参数作为请求的正文，它不可以以"{"开头：

```
curl -XGET 'localhost:9200/_analyze?tokenizer=keyword&filter=lowercase&char_filter=reverse' -d 'this is a test' -H 'Content-Type: text/plain'
```

###### 警告

> 在5.1.0中已被弃用，不推荐使用text参数作为请求正文，并且此功能将在下一个主要版本中删除。 请使用JSON text参数。

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/5.4/indices-analyze.html
