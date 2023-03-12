---
title: "ElasticSearch官网翻译_MatchPhrasePrefixQuery"
date: 2017-06-17 15:48:04
tags: [Elasticsearch]
categories: [Search]
---

## Match Phrase Prefix Query（短语前缀匹配查询）

match_phrase_prefix与match_phrase相同，除了它允许文本中最后一个词条的前缀匹配。 例如：

```
GET /_search
{
    "query": {
        "match_phrase_prefix" : {
            "message" : "quick brown f"
        }
    }
}
```

它接受与短语类型相同的参数。 此外，它还接受一个max_expansions参数（默认50），可以控制最后一个词条将被扩展多少个后缀。 强烈建议将其设置为可接受的值以控制查询的执行时间。 例如：

```
GET /_search
{
    "query": {
        "match_phrase_prefix" : {
            "message" : {
                "query" : "quick brown f",
                "max_expansions" : 10
            }
        }
    }
}
```

###### 重要

> match_phrase_prefix查询是一个糟糕的autocomplete（自动填充）。 这是非常容易使用的，这可以让你快速开始search-as-you-type（输入即搜索），但其结果通常足够好，有时可能会令人困惑。

> 考虑查询字符串"quick brown f"。 此查询的工作原理是通过quick和brown创建短语查询（即词条quick必须存在，并且它之后必须是词条brown）。 然后，它查看排序的词条字典，以查找以"f"开头的前50个词条，并将这些词条添加到短语查询中。

> 问题是，前50个词条可能不包含词条fox，所以短语"quick brown fox"没有被找到。 这通常不是问题，因为用户将继续输入更多的字母，直到他们正在寻找的词出现。

> 为了更好的search-as-you-type（搜索你的输入）解决方案，请参阅completion suggester（补全提示）和Index-Time Search-as-You-Type（索引时输入即搜索）。

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/5.4/query-dsl-match-query-phrase-prefix.html
