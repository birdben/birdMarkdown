---
title: "ElasticSearch官网翻译_DisMaxQuery"
date: 2017-06-29 20:10:44
tags: [Elasticsearch]
categories: [Search]
---

## Dis Max Query

此查询生成由其子查询生成的文档的并集，并且计算每个文档的得分使用任何子查询生成的文档的最大分数，加上其他匹配子查询的分数增量（tie breaking属性）。

这很有用，当在具有不同boost因子的多个字段中搜索单词时（这样字段不能被等效地组合成单个搜索字段）。我们希望主分数与最高boost相关联，而不是字段分数的总和（如Boolean Query给出的）。如果查询是"albino elephant"，则可以确保匹配一个字段的"albino"和另一个匹配的"elephant"比两个字段都匹配的"albino"更高。要获得此结果，请使用Boolean Query和DisjunctionMax Query：DisjunctionMaxQuery在每个字段中搜索每个词条，而将这些DisjunctionMaxQuery的集合组合成BooleanQuery。

tie breaker能力在于允许在多个字段中包括相同词条的结果，比仅在这些多个字段中的最佳字段包括该词条的结果更好地判断，而不会将这与多个字段中的两个不同词条的更好的情况混淆。 tie_breaker的默认值为0.0。

此查询映射到Lucene DisjunctionMaxQuery。

```
GET /_search
{
    "query": {
        "dis_max" : {
            "tie_breaker" : 0.7,
            "boost" : 1.2,
            "queries" : [
                {
                    "term" : { "age" : 34 }
                },
                {
                    "term" : { "age" : 35 }
                }
            ]
        }
    }
}
```

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/5.4/query-dsl-dis-max-query.html
