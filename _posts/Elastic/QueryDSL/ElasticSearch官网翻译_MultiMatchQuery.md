---
title: "ElasticSearch官网翻译_MultiMatchQuery"
date: 2017-06-17 17:43:36
tags: [Elasticsearch]
categories: [Search]
---

## Multi Match Query（多匹配查询）

multi_match查询基于match查询构建，以允许multi-field多字段查询：

```
GET /_search
{
  "query": {
    "multi_match" : {
      "query":    "this is a test", (1)
      "fields": [ "subject", "message" ] (2)
    }
  }
}
```

- (1) 查询字符串。
- (2) 要查询的字段。

#### fields and per-field boosting（fields和每个field的提升）

可以使用通配符指定字段，例如：

```
GET /_search
{
  "query": {
    "multi_match" : {
      "query":    "Will Smith",
      "fields": [ "title", "*_name" ] (1)
    }
  }
}
```

- (1) 查询title，first_name和last_name字段

个别字段可以用插入符号（^）表示法来提升：

```
GET /_search
{
  "query": {
    "multi_match" : {
      "query" : "this is a test",
      "fields" : [ "subject^3", "message" ] (1)
    }
  }
}
```

- (1) subject的重要程度是message的3倍

#### Types of multi_match query（multi_match查询类型）

内部执行multi_match查询的方式取决于type参数，可以将其设置为：

值|描述
---|---
best_fields|（默认）查找与任何字段匹配的文档，但使用最佳字段中的_score。 查看best_fields。
most_fields|查找与任何字段匹配的文档，并组合每个字段的_score。 查看most_fields。
cross_fields|使用相同的分析器来处理字段，就像他们是一个大字段一样。 寻找任何字段中的每个单词。 查看cross_fields。
phrase|在每个字段上运行一个match_phrase查询，并组合每个字段的_score。 查看phrase and phrase_prefix。
phrase_prefix|在每个字段上运行一个match_phrase_prefix查询，并组合每个字段的_score。 查看phrase and phrase_prefix。

### best_fields

当你搜索多个单词在同一字段中的最佳结果时，best_fields类型最为有用。 例如，"brown fox"在同一个字段，比"brown"在一个字段"fox"在另一个字段更有意义。

best_fields类型为每个字段生成match查询，并将其包装在dis_max查询中，以找到单个最佳匹配字段。 例如，这个查询：

```
GET /_search
{
  "query": {
    "multi_match" : {
      "query":      "brown fox",
      "type":       "best_fields",
      "fields":     [ "subject", "message" ],
      "tie_breaker": 0.3
    }
  }
}
```

将被执行为：

```
GET /_search
{
  "query": {
    "dis_max": {
      "queries": [
        { "match": { "subject": "brown fox" }},
        { "match": { "message": "brown fox" }}
      ],
      "tie_breaker": 0.3
    }
  }
}
```

通常，best_fields类型使用单个最佳匹配字段的分数，但是如果指定了tie_breaker，则它计算得分如下：

- 使用最佳匹配字段的得分
- 加上所有其他匹配字段的tie_breaker * _score

此外，如match query中所述接受analyzer，boost，operator，minimum_should_match，fuzziness，lenient，prefix_length，max_expansions，rewrite，zero_terms_query和cutoff_frequency。

###### 重要：operator和minimum_should_match

> best_fields和most_fields类型是field-centric（以字段为中心的） - 它们会为每个字段生成match查询。 这意味着operator和minimum_should_match参数将单独应用于每个字段，这可能不是你想要的。
> 
> 以此查询为例：
>
```
GET /_search
{
  "query": {
    "multi_match" : {
      "query":      "Will Smith",
      "type":       "best_fields",
      "fields":     [ "first_name", "last_name" ],
      "operator":   "and" (1)
    }
  }
}
```
> 
> (1) 所有词条必须存在。
>
> 该查询执行如下：
> 
```
(+first_name:will +first_name:smith)
| (+last_name:will  +last_name:smith)
```
> 
> 换句话说，所有词条都必须存在于单个字段中以供文档匹配。
>
> 请参阅cross_fields以获得更好的解决方案。

### most_fields

当查询多个字段包含以不同方式分析的相同文本时，most_fields类型最为有用。 例如，主要字段可能包含synonyms（同义词），stemming（词干）和terms without diacritics（没有变音符号的词条）。 第二个字段可能包含original terms（原始词条），第三个字段可能包含shingles（小圆石？）。 通过组合来自所有三个字段的分数，我们可以将与主要字段尽可能多的文档相匹配，但使用第二和第三字段将最相似的结果推送到列表的顶部。

该查询：

```
GET /_search
{
  "query": {
    "multi_match" : {
      "query":      "quick brown fox",
      "type":       "most_fields",
      "fields":     [ "title", "title.original", "title.shingles" ]
    }
  }
}
```

该查询执行如下：

```
GET /_search
{
  "query": {
    "bool": {
      "should": [
        { "match": { "title":          "quick brown fox" }},
        { "match": { "title.original": "quick brown fox" }},
        { "match": { "title.shingles": "quick brown fox" }}
      ]
    }
  }
}
```

每个match子句的得分加在一起，然后除以match子句的数目。

另外，接受analyzer，boost，operator，minimum_should_match，fuzziness，lenient，prefix_length，max_expansions，rewrite，zero_terms_query和cutoff_frequency，如match query中所述，但请参阅operator和minimum_should_match。

### phrase and phrase_prefix

phrase和phrase_prefix类型的行为就像best_fields一样，但是它们使用match_phrase或match_phrase_prefix查询，而不是match查询。

该查询：

```
GET /_search
{
  "query": {
    "multi_match" : {
      "query":      "quick brown f",
      "type":       "phrase_prefix",
      "fields":     [ "subject", "message" ]
    }
  }
}
```

该查询执行如下：

```
GET /_search
{
  "query": {
    "dis_max": {
      "queries": [
        { "match_phrase_prefix": { "subject": "quick brown f" }},
        { "match_phrase_prefix": { "message": "quick brown f" }}
      ]
    }
  }
}
```

另外，如Match Query中所述接受analyzer，boost，lenient，slop和zero_terms_query。phrase_prefix类型额外接受max_expansions。

###### 重要：phrase, phrase_prefix and fuzziness

> fuzziness参数不能与phrase或phrase_prefix类型一起使用。

### cross_fields

cross_fields类型对于多个字段应匹配的结构化文档特别有用。 例如，当查询"Will Smith"的first_name和last_name字段时，最佳匹配是在一个字段中可能具有"Will"，而在另一个字段中可能具有"Smith"。

> 这听起来像是most_fields的工作，但是这种方法有两个问题。 第一个问题是operator和minimum_should_match应用于per-field（每个字段），而不是per-term（每个词条）（参见上面的说明）。
>
> 第二个问题是与相关性有关：first_name和last_name字段中的不同词条频率可能会产生意想不到的结果。
>
> 例如，想象一下我们有两个人："Will Smith"和"Smith Jones"。 "Smith"作为一个last name是非常普遍的（因此是不太重要的），但"Smith"作为一个first name是非常罕见的（这是非常重要的）。
>
> 如果我们搜索"Will Smith"，"Smith Jones"文档可能会出现在更好的匹配"Will Smith"之上，因为first_name:smith的得分胜过了first_name:will加上last_name:smith的得分。

处理这些类型查询的一种方法是将first_name和last_name字段索引到一个full_name字段。 当然，这只能在索引时完成。

cross_field类型尝试通过采用term-centric（以词条为中心）的方法在查询时解决这些问题。 它首先将查询字符串分析为单个词条，然后在任何字段中查找每个词条，就像它们是一个大字段一样。

查询如下：

```
GET /_search
{
  "query": {
    "multi_match" : {
      "query":      "Will Smith",
      "type":       "cross_fields",
      "fields":     [ "first_name", "last_name" ],
      "operator":   "and"
    }
  }
}
```

被执行如下：

```
+(first_name:will  last_name:will)
+(first_name:smith last_name:smith)
```

换句话说，所有词条必须至少存在一个字段以匹配文档。 （将此与用于best_fields和most_fields的逻辑进行比较）

这解决了两个问题之一。 通过混合所有字段的词条频率来解决不同词条频率的问题，以便平衡差异。

在实践中，first_name:smith将被视为与last_name:smith加一具有相同的频率。 这将使first_name和last_name上的匹配具有相似的分数，对于last_name来说，这是一个微弱的优势，因为它是包含smith的最有可能的字段。

请注意，cross_fields通常仅适用于所有boost为1的短字符串字段。否则，boosts，term freqs和length标准化将影响计算得分，使得词条统计的混合不再有意义。

如果你通过验证API运行上述查询，则返回此说明：

```
+blended("will",  fields: [first_name, last_name])
+blended("smith", fields: [first_name, last_name])
```

另外，接受analyzer，boost，operator，minimum_should_match，lenient，zero_terms_query和cutoff_frequency，如match query中所述。

##### cross_field and analysis

cross_field类型只能在具有相同分析器的字段上term-centric（以词条为中心）的模式工作。 具有相同分析器的字段被组合在一起如上述示例。 如果有多个组，则它们与bool查询相结合。

例如，如果我们有first和last字段具有相同的分析器，再加上使用edge_ngram分析器的first.edge和last.edge两个字段，这个查询：

```
GET /_search
{
  "query": {
    "multi_match" : {
      "query":      "Jon",
      "type":       "cross_fields",
      "fields":     [
        "first", "first.edge",
        "last",  "last.edge"
      ]
    }
  }
}
```

被执行如下：

```
    blended("jon", fields: [first, last])
| (
    blended("j",   fields: [first.edge, last.edge])
    blended("jo",  fields: [first.edge, last.edge])
    blended("jon", fields: [first.edge, last.edge])
)
```

换句话说，first和last将被分组在一起并被视为单个字段，并且first.edge和last.edge将被分组在一起并被视为单个字段。

拥有多个组是好的，但是当与operator或minimum_should_match组合时，它可能遭受与most_fields或best_fields相同的问题。

你可以轻松地将此查询重写为两个独立的cross_fields查询与bool查询相结合，并将minimum_should_match参数应用于其中一个：

```
GET /_search
{
  "query": {
    "bool": {
      "should": [
        {
          "multi_match" : {
            "query":      "Will Smith",
            "type":       "cross_fields",
            "fields":     [ "first", "last" ],
            "minimum_should_match": "50%" (1)
          }
        },
        {
          "multi_match" : {
            "query":      "Will Smith",
            "type":       "cross_fields",
            "fields":     [ "*.edge" ]
          }
        }
      ]
    }
  }
}
```

- (1) will或者smith都必须存在于first或last字段中

你可以通过在查询中指定analyzer参数来强制所有字段进入同一组。

```
GET /_search
{
  "query": {
   "multi_match" : {
      "query":      "Jon",
      "type":       "cross_fields",
      "analyzer":   "standard", 
      "fields":     [ "first", "last", "*.edge" ]
    }
  }
}
```

- (1) 对所有字段使用standard标准分析器。

被执行如下：

```
blended("will",  fields: [first, first.edge, last.edge, last])
blended("smith", fields: [first, first.edge, last.edge, last])
```

##### tie_breaker

默认情况下，每个词条blended查询将使用组中任何字段返回的最佳分数，然后将这些分数相加，以得出最终得分。 tie_breaker参数可以更改每个词条的blended查询的默认行为。 它接受：

值|描述
---|---
0.0|取一个最好的成绩（例如）first_name:will和last_name:will（默认）
1.0|将分数加分（例如）first_name:will和last_name:will
0.0 < n < 1.0|以单个最佳得分加上tie_breaker乘以其他匹配字段的每个分数。

###### 重要：cross_fields and fuzziness

> fuzziness参数不能与cross_fields类型一起使用。

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/5.4/query-dsl-multi-match-query.html
