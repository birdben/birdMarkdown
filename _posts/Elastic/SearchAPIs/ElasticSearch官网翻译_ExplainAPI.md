---
title: "ElasticSearch官网翻译_ExplainAPI"
date: 2017-05-28 12:32:34
tags: [Elasticsearch]
categories: [Search]
---

#### Explain API

解释api计算查询和特定文档的分数说明。 这可以给出文档是否与特定查询匹配或不匹配的有用反馈。

索引和类型参数分别期望单个索引和单个类型。

##### 用法

完整查询示例：

```
curl -XGET 'localhost:9200/twitter/tweet/1/_explain' -d '{
      "query" : {
        "term" : { "message" : "search" }
      }
}'
```

这将产生以下结果：

```
{
  "matches" : true,
  "explanation" : {
    "value" : 0.15342641,
    "description" : "fieldWeight(message:search in 0), product of:",
    "details" : [ {
      "value" : 1.0,
      "description" : "tf(termFreq(message:search)=1)"
    }, {
      "value" : 0.30685282,
      "description" : "idf(docFreq=1, maxDocs=1)"
    }, {
      "value" : 0.5,
      "description" : "fieldNorm(field=message, doc=0)"
    } ]
  }
}
```

还有一种通过q参数指定查询的更简单的方法。 然后解析指定的q参数值，就像使用query_string查询一样。 在解释api中使用q参数的示例：

```
curl -XGET 'localhost:9200/twitter/tweet/1/_explain?q=message:search'
```

这将产生与先前请求相同的结果。

##### All parameters

_source|设置为true以取回解释的文档的_source。 你还可以使用_source_include＆_source_exclude取回文档的一部分（有关详细信息，请参阅Get API）
fields|允许控制哪些存储字段作为解释的文档的一部分返回。
routing|在索引期间使用路由的情况下控制路由。
parent|与设置路由参数相同的效果。
preference|控制执行解释的分片。
source|允许请求的数据放在url的query string查询字符串中。
q|查询字符串（映射到query_string查询）。
df|在查询中未定义字段前缀时使用的默认字段。 默认为_all字段。
analyzer|分析查询字符串时使用的分析器名称。 默认为_all字段的分析器。
analyze_wildcard|应该分析通配符和前缀查询。 默认为false。
lowercase_expanded_terms|词条是否应该自动小写化。 默认为true。
lenient|如果设置为true，将会导致基于格式的失败（例如提供文本到数字字段）被忽略。 默认为false。
default_operator|要使用的默认运算符可以是AND或OR。 默认为OR。

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/2.4/search-explain.html

