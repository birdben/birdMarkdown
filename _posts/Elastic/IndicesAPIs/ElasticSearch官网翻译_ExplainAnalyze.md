---
title: "ElasticSearch官网翻译_ExplainAnalyze"
date: 2017-06-05 15:05:07
tags: [Elasticsearch]
categories: [Search]
---

## Explain Analyze

如果要获取更多高级细节，请将explain设置为true（默认为false）。 它将为每个标记输出所有token标记属性。 你可以通过设置属性选项过滤要输出的标记属性。

###### 警告

> 附加详细信息的格式是处于试验阶段的，可随时更改

```
GET _analyze
{
  "tokenizer" : "standard",
  "filter" : ["snowball"],
  "text" : "detailed output",
  "explain" : true,
  "attributes" : ["keyword"] (1)
}
```

- (1) 设置"keyword"仅输出"keyword"属性

请求返回以下结果：

```
{
  "detail" : {
    "custom_analyzer" : true,
    "charfilters" : [ ],
    "tokenizer" : {
      "name" : "standard",
      "tokens" : [ {
        "token" : "detailed",
        "start_offset" : 0,
        "end_offset" : 8,
        "type" : "<ALPHANUM>",
        "position" : 0
      }, {
        "token" : "output",
        "start_offset" : 9,
        "end_offset" : 15,
        "type" : "<ALPHANUM>",
        "position" : 1
      } ]
    },
    "tokenfilters" : [ {
      "name" : "snowball",
      "tokens" : [ {
        "token" : "detail",
        "start_offset" : 0,
        "end_offset" : 8,
        "type" : "<ALPHANUM>",
        "position" : 0,
        "keyword" : false 
      }, {
        "token" : "output",
        "start_offset" : 9,
        "end_offset" : 15,
        "type" : "<ALPHANUM>",
        "position" : 1,
        "keyword" : false 
      } ]
    } ]
  }
}
```

- (1)(2) 仅输出"keyword"属性，因为在请求中指定"attributes"。

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/5.4/_explain_analyze.html
