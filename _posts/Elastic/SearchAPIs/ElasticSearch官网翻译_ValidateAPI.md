---
title: "ElasticSearch官网翻译_ValidateAPI"
date: 2017-05-28 12:18:00
tags: [Elasticsearch]
categories: [Search]
---

#### Validate API

Validate API允许用户验证潜在昂贵的查询而不执行它。 以下示例显示如何使用它：

```
curl -XPUT 'http://localhost:9200/twitter/tweet/1' -d '{
    "user" : "kimchy",
    "post_date" : "2009-11-15T14:12:12",
    "message" : "trying out Elasticsearch"
}'
```

当查询有效时，响应包含valid:true

```
curl -XGET 'http://localhost:9200/twitter/_validate/query?q=user:foo'
{"valid":true,"_shards":{"total":1,"successful":1,"failed":0}}
```

##### Request Parameters

当使用查询参数q执行时，传递的查询是使用Lucene查询解析器的查询字符串。 还有其他可以传递的参数：

Name|Description
---|---
df|在查询中未定义字段前缀时使用的默认字段。
analyzer|分析查询字符串时使用的分析器名称。
default_operator|要使用的默认运算符可以是AND或OR。 默认为OR。
lenient|如果设置为true，将会导致基于格式的失败（例如提供文本到数字字段）被忽略。 默认为false。
lowercase_expanded_terms|词条是否应该自动小写化。 默认为true。
analyze_wildcard|应该分析通配符和前缀查询。 默认为false。

或者，请求体：

```
curl -XGET 'http://localhost:9200/twitter/tweet/_validate/query' -d '{
  "query" : {
    "bool" : {
      "must" : {
        "query_string" : {
          "query" : "*:*"
        }
      },
      "filter" : {
        "term" : { "user" : "kimchy" }
      }
    }
  }
}'
{"valid":true,"_shards":{"total":1,"successful":1,"failed":0}}
```

###### 注意

在主体中发送的查询必须嵌套在query key中，与search API相同

如果查询无效的，则valid将为false。 这里的查询是无效的，因为Elasticsearch知道基于动态映射post_date字段应该是日期，而foo没有正确解析成日期：

```
curl -XGET 'http://localhost:9200/twitter/tweet/_validate/query?q=post_date:foo'
{"valid":false,"_shards":{"total":1,"successful":1,"failed":0}}
```

可以指定一个explain参数来获取有关查询失败原因的更多详细信息：

```
curl -XGET 'http://localhost:9200/twitter/tweet/_validate/query?q=post_date:foo&pretty=true&explain=true'
{
  "valid" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "failed" : 0
  },
  "explanations" : [ {
    "index" : "twitter",
    "valid" : false,
    "error" : "[twitter] QueryParsingException[Failed to parse]; nested: IllegalArgumentException[Invalid format: \"foo\"];; java.lang.IllegalArgumentException: Invalid format: \"foo\""
  } ]
}
```

当查询有效时，explanation详细说明将默认为该查询的字符串表示形式。 将rewrite设置为true时，explanation详细说明将更详细地显示将要执行的实际Lucene查询。

对于模糊查询：

```
curl -XGET 'http://localhost:9200/imdb/movies/_validate/query?rewrite=true' -d '
{
  "query": {
    "fuzzy": {
      "actors": "kyle"
    }
  }
}'
```

响应：

```
{
   "valid": true,
   "_shards": {
      "total": 1,
      "successful": 1,
      "failed": 0
   },
   "explanations": [
      {
         "index": "imdb",
         "valid": true,
         "explanation": "plot:kyle plot:kylie^0.75 plot:kyne^0.75 plot:lyle^0.75 plot:pyle^0.75 #_type:movies"
      }
   ]
}
```

更多如此：

```
curl -XGET 'http://localhost:9200/imdb/movies/_validate/query?rewrite=true'
{
  "query": {
    "more_like_this": {
      "like": {
        "_id": "88247"
      },
      "boost_terms": 1
    }
  }
}
```

响应：

```
{
   "valid": true,
   "_shards": {
      "total": 1,
      "successful": 1,
      "failed": 0
   },
   "explanations": [
      {
         "index": "imdb",
         "valid": true,
         "explanation": "((title:terminator^3.71334 plot:future^2.763601 plot:human^2.8415773 plot:sarah^3.4193945 plot:kyle^3.8244398 plot:cyborg^3.9177752 plot:connor^4.040236 plot:reese^4.7133346 ... )~6) -ConstantScore(_uid:movies#88247) #_type:movies"
      }
   ]
}
```

###### 警告

该请求仅在单个分片上执行，这是随机选择的。 查询的explanation详细说明可能取决于哪个分片被命中，因此可能会因请求而异。

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/2.4/search-validate.html

