---
title: "ElasticSearch官网翻译_MatchQuery"
date: 2017-06-16 16:55:03
tags: [Elasticsearch]
categories: [Search]
---

## Match Query

match查询接受text/numerics/dates，分析它们，并构造查询。 例如：

```
GET /_search
{
    "query": {
        "match" : {
            "message" : "this is a test"
        }
    }
}
```

注意，message是一个字段的名称，可以替换任何字段的名称（包括_all）。

### match（匹配查询）

match查询的类型为boolean。 这意味着分析所提供的文本，并且分析过程从提供的文本构建布尔查询。 可以将operator（运算符）标志设置为or或and以控制布尔子句（默认为or）。 可以使用minimum_should_match参数设置要匹配的可选should子句的最小数量。

analyzer可以设置为控制哪个分析器将对文本执行分析过程。 它默认为字段显式映射定义，或默认search analyzer（搜索分析器）。

可以将lenient参数设置为true以忽略由数据类型不匹配引起的异常，例如尝试使用文本查询字符串查询数字字段。 默认为false。

#### Fuzziness（模糊查询）

fuzziness允许基于被查询的字段的类型进行fuzzy matching（模糊匹配）。 有关允许的设置，请参阅"Fuzziness"一节。

在这种情况下，可以设置prefix_length和max_expansions来控制模糊过程。 如果设置了模糊选项，则查询将使用top_terms_blended_freqs_${max_expansions}作为其rewrite method（重写方法），fuzzy_rewrite参数允许控制查询将如何重写。

模糊转换（ab→ba）默认允许，但可以通过将fuzzy_transposition设置为false来禁用。

以下是提供附加参数的一个例子（注意结构略有变化，message是字段名称）：

```
GET /_search
{
    "query": {
        "match" : {
            "message" : {
                "query" : "this is a test",
                "operator" : "and"
            }
        }
    }
}
```

#### Zero terms query（零词条查询）

如果使用的分析器删除了查询（像stop过滤器这样）中的所有tokens（词元），则默认行为是根本不匹配任何文档。 为了更改，可以使用zero_terms_query选项，它接受none（默认值）和all，相当于match_all查询。

```
GET /_search
{
    "query": {
        "match" : {
            "message" : {
                "query" : "to be or not to be",
                "operator" : "and",
                "zero_terms_query": "all"
            }
        }
    }
}
```

#### Cutoff frequency（截断频率）

match查询支持cutoff_frequency，允许指定绝对或相对文档频率，其中高频项被移动到可选子查询中，并且如果是or操作符匹配则只计算一个低频（低于cutoff）项，或者如果是and操作符匹配则计算全部的低频项。

此查询允许在运行时动态处理stopwords（停用词），与域无关，不需要stopword文件。它可以防止对高频词条进行评分/迭代，并且如果更重要/更低的频率词条与文档匹配，则仅考虑这些词条。然而，如果所有查询词条都高于给定的cutoff_frequency，则查询将自动转换为纯连接（and）查询，以确保快速执行。

如果在[0..1]范围内，则cut_frequency可以相对于文档的总数，如果大于或等于1.0，则可以是absolute_frequency。

下面是一个例子，显示了一个仅仅由stopwords组成的查询：

```
GET /_search
{
    "query": {
        "match" : {
            "message" : {
                "query" : "to be or not to be",
                "cutoff_frequency" : 0.001
            }
        }
    }
}
```

###### 重要

> cutoff_frequency选项以per-shard-level（每个分片级别）运行。 这意味着当你尝试使用少量文档数测试索引时，你应该遵循Relevance is broken中的建议。

####### Comparison to query_string/field（比较query_string/field）

> match查询系列不会通过“query parsing（查询解析）”过程。 它不支持字段名称前缀，通配符或其他“高级”功能。 由于这个原因，它失败的机会非常小/不存在，并且它提供了一个很好的行为，当它仅仅是分析和运行该文本作为查询行为（通常是一个文本搜索框）时。 此外，phrase_prefix类型可以提供一个很好的“你输入”行为来自动加载搜索结果。 


参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/5.4/query-dsl-match-query.html
