---
title: "ElasticSearch官网翻译_MultiTermvectorsAPI"
date: 2017-06-01 19:11:44
tags: [Elasticsearch]
categories: [Search]
---

## Multi termvectors API

Multi termvectors API允许一次获得多个词条向量。 取回的词条向量的文档可以由index，type和id指定。 但文档也可以在请求中由人为提供。

响应包括一个具有所有获取的词条向量的docs数组，每个元素具有由termvectors API提供的结构。 这是一个例子：

```
POST /_mtermvectors
{
   "docs": [
      {
         "_index": "twitter",
         "_type": "tweet",
         "_id": "2",
         "term_statistics": true
      },
      {
         "_index": "twitter",
         "_type": "tweet",
         "_id": "1",
         "fields": [
            "message"
         ]
      }
   ]
}
```

有关可能的参数的描述，请参阅termvectors API。

_mtermvectors endpoint也可以针对索引使用（在这种情况下，_index不需要在请求主体中）：

```
POST /twitter/_mtermvectors
{
   "docs": [
      {
         "_type": "tweet",
         "_id": "2",
         "fields": [
            "message"
         ],
         "term_statistics": true
      },
      {
         "_type": "tweet",
         "_id": "1"
      }
   ]
}
```

type同理：

```
POST /twitter/tweet/_mtermvectors
{
   "docs": [
      {
         "_id": "2",
         "fields": [
            "message"
         ],
         "term_statistics": true
      },
      {
         "_id": "1"
      }
   ]
}
```

如果所有请求的文档都在相同的索引并且具有相同的类型，并且参数是相同的，则可以简化请求：

```
POST /twitter/tweet/_mtermvectors
{
    "ids" : ["1", "2"],
    "parameters": {
        "fields": [
                "message"
        ],
        "term_statistics": true
    }
}
```

此外，就像对于termvectors API一样，可以为用户提供的文档生成词条向量。 所使用的映射由_index和_type确定。

```
POST /_mtermvectors
{
   "docs": [
      {
         "_index": "twitter",
         "_type": "tweet",
         "doc" : {
            "user" : "John Doe",
            "message" : "twitter test test test"
         }
      },
      {
         "_index": "twitter",
         "_type": "test",
         "doc" : {
           "user" : "Jane Doe",
           "message" : "Another twitter test ..."
         }
      }
   ]
}
```

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/5.4/docs-multi-termvectors.html
