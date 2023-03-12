---
title: "ElasticSearch官网翻译_CommonTermsQuery"
date: 2017-06-18 12:33:19
tags: [Elasticsearch]
categories: [Search]
---

## Common Terms Query

Common Terms Query是一种最新的替代方法，可以提高搜索结果的精确度和召回（通过考虑停用词），而不会牺牲性能。

### 问题

查询中的每个词条都有成本。 搜索"The brown fox"需要三个term查询，"the"，"brown"和"fox"每个查询都针对索引中的所有文档执行。 对"the"的查询可能匹配许多文档，因此相对于其他两个词条的影响要小得多。

以前，解决这个问题的办法是忽视高频率的词条。 通过将"the"视为一个stopword（停止词），我们减少索引大小并减少需要执行的term查询的数量。

这种方法的问题是，尽管stopwords（停止词）对相关性的影响很小，但它们仍然很重要。 如果我们删除stopwords，我们会失去精确度（例如，我们无法区分"happy"和"not happy"），我们失去了召回（例如像"The The"或"To be or not to be"这样的文本根本不会存在于索引中）。

### 解决办法

Common Terms Query将查询词条分为两组：更重要的（即低频词条）和较不重要的（即之前是stopwords的高频词条）。

首先，它搜索符合更重要词条的文档。这些词条在较少的文档中出现，对相关性的影响更大。

然后，它对较不重要的词条执行第二个查询 - 频繁出现且对相关性影响较小的词条。但是，除了计算所有匹配文档的相关性分数之外，它只计算已经与第一个查询匹配的文档的_score。以这种方式，高频率词条可以改善相关性计算，而不需要支付性能差的成本。

如果查询仅由高频词条组成，则单个查询将作为AND（连接）查询执行，换句话说，所有词条都是必须（匹配）的。即使每个单独的词条将匹配许多文档，词条的组合将缩小结果集合以保留最相关的。单个查询也可以使用"OR"执行并指定minimum_should_match，在这种情况下应该使用足够高的值。

词条被分配到高频词条组或者低频词条组取决于cutoff_frequency，cutoff_frequency可以被指定为绝对频率（> = 1）或相对频率（0.0 .. 1.0）。 （请记住，文档频率按照分片级别进行计算，如在Relevance is broken文章中所述。）

也许这个查询最有趣的属性是它自动适应领域使用特定的停用词。例如，在视频托管网站上，诸如"clip"或"video"之类的常用词条将自动表现为stopwords，而无需维护手动列表。

### 例子

在这个例子中，文档频率大于0.1％的单词（例如"this"和"is"）将被视为common terms（常用词条）。

```
GET /_search
{
    "query": {
        "common": {
            "body": {
                "query": "this is bonsai cool",
                    "cutoff_frequency": 0.001
            }
        }
    }
}
```

可以使用minimum_should_match（high_freq，low_freq），low_freq_operator（默认"or"）和high_freq_operator（默认"or"）参数来控制应该匹配的词条数量。

对于低频词条，将low_freq_operator设置为"and"以使所有（低频）词条都是必须（匹配）的：

```
GET /_search
{
    "query": {
        "common": {
            "body": {
                "query": "nelly the elephant as a cartoon",
                    "cutoff_frequency": 0.001,
                    "low_freq_operator": "and"
            }
        }
    }
}
```

大致相当于：

```
GET /_search
{
    "query": {
        "bool": {
            "must": [
            { "term": { "body": "nelly"}},
            { "term": { "body": "elephant"}},
            { "term": { "body": "cartoon"}}
            ],
            "should": [
            { "term": { "body": "the"}},
            { "term": { "body": "as"}},
            { "term": { "body": "a"}}
            ]
        }
    }
}
```

或者，使用minimum_should_match指定必须存在的低频词条的最小数量或百分比，例如：

```
GET /_search
{
    "query": {
        "common": {
            "body": {
                "query": "nelly the elephant as a cartoon",
                "cutoff_frequency": 0.001,
                "minimum_should_match": 2
            }
        }
    }
}
```

大致相当于：

```
GET /_search
{
    "query": {
        "bool": {
            "must": {
                "bool": {
                    "should": [
                    { "term": { "body": "nelly"}},
                    { "term": { "body": "elephant"}},
                    { "term": { "body": "cartoon"}}
                    ],
                    "minimum_should_match": 2
                }
            },
            "should": [
                { "term": { "body": "the"}},
                { "term": { "body": "as"}},
                { "term": { "body": "a"}}
                ]
        }
    }
}
```

minimum_should_match

可以使用附加的low_freq和high_freq参数对低频词条和高频词条应用一个不同的minimum_should_match。 以下是提供附加参数的示例（请注意结构的更改）：

```
GET /_search
{
    "query": {
        "common": {
            "body": {
                "query": "nelly the elephant not as a cartoon",
                    "cutoff_frequency": 0.001,
                    "minimum_should_match": {
                        "low_freq" : 2,
                        "high_freq" : 3
                    }
            }
        }
    }
}
```

大致相当于：

```
GET /_search
{
    "query": {
        "bool": {
            "must": {
                "bool": {
                    "should": [
                    { "term": { "body": "nelly"}},
                    { "term": { "body": "elephant"}},
                    { "term": { "body": "cartoon"}}
                    ],
                    "minimum_should_match": 2
                }
            },
            "should": {
                "bool": {
                    "should": [
                    { "term": { "body": "the"}},
                    { "term": { "body": "not"}},
                    { "term": { "body": "as"}},
                    { "term": { "body": "a"}}
                    ],
                    "minimum_should_match": 3
                }
            }
        }
    }
}
```

在这种情况下，这意味着高频率词条（至少满足他们中同时出现三个）才对相关性产生影响。 但是，对于高频率词条来说，minimum_should_match最有趣的使用方法是（查询字符串中）只有高频词条：

```
GET /_search
{
    "query": {
        "common": {
            "body": {
                "query": "how not to be",
                    "cutoff_frequency": 0.001,
                    "minimum_should_match": {
                        "low_freq" : 2,
                        "high_freq" : 3
                    }
            }
        }
    }
}
```

大致相当于：

```
GET /_search
{
    "query": {
        "bool": {
            "should": [
            { "term": { "body": "how"}},
            { "term": { "body": "not"}},
            { "term": { "body": "to"}},
            { "term": { "body": "be"}}
            ],
            "minimum_should_match": "3<50%"
        }
    }
}
```

然后，高频生成查询的限制比AND更少。

Common Terms Query还支持boost，analyzer和disable_coord作为参数。

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/5.4/query-dsl-common-terms-query.html
