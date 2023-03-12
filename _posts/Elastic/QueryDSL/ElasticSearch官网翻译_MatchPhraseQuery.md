---
title: "ElasticSearch官网翻译_MatchQuery"
date: 2017-06-17 14:23:24
tags: [Elasticsearch]
categories: [Search]
---

## Match Phrase Query（短语匹配查询）

match_phrase查询分析文本，并从分析文本中创建phrase（短语）查询。 例如：

```
GET /_search
{
    "query": {
        "match_phrase" : {
            "message" : "this is a test"
        }
    }
}
```

phrase（短语）查询以任何顺序将词条匹配到可配置的slop（默认为0）。 使调换顺序的词条的slop为2。

analyzer可以设置为控制哪个分析器将对文本执行分析过程。 它默认为字段显式映射定义，或默认search analyzer（搜索分析器），例如：

```
GET /_search
{
    "query": {
        "match_phrase" : {
            "message" : {
                "query" : "this is a test",
                "analyzer" : "my_analyzer"
            }
        }
    }
}
```

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/5.4/query-dsl-match-query-phrase.html
