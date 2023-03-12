---
title: "ElasticSearch官网翻译_MoreLikeThisQuery"
date: 2017-07-12 14:49:21
tags: [Elasticsearch]
categories: [Search]
---

## More Like This Query

More Like This Query（MLT Query）可以查找"like"一组文档的文档。 为了做到这一点，MLT选择这些输入文档的一组代表性词条，使用这些词条形成查询，执行查询并返回结果。 用户控制输入文档，如何选择词条以及如何形成查询。 more_like_this可以缩短为mlt。（[5.0.0] 5.0.0中弃用。 使用more_like_this来代替）

最简单的用例包括查找与提供的文本片段类似的文档。 在这里，我们要查找所有的电影都有类似于"Once upon a time"在他们的"title"和"description"字段中，将所选的词条数量限制为12。

```
GET /_search
{
    "query": {
        "more_like_this" : {
            "fields" : ["title", "description"],
            "like" : "Once upon a time",
            "min_term_freq" : 1,
            "max_query_terms" : 12
        }
    }
}
```

更复杂的用例包括将文本与索引中已经存在的文档进行混合。 在这种情况下，指定文档的语法与Multi GET API中使用的语法相似。

```
GET /_search
{
    "query": {
        "more_like_this" : {
            "fields" : ["title", "description"],
            "like" : [
            {
                "_index" : "imdb",
                "_type" : "movies",
                "_id" : "1"
            },
            {
                "_index" : "imdb",
                "_type" : "movies",
                "_id" : "2"
            },
            "and potentially some more text here as well"
            ],
            "min_term_freq" : 1,
            "max_query_terms" : 12
        }
    }
}
```

最后，用户可以混合一些文本，一组选定的文档，也可以提供索引中不一定存在的文档。 要提供索引中不存在的文档，语法类似于artificial documents。

```
GET /_search
{
    "query": {
        "more_like_this" : {
            "fields" : ["name.first", "name.last"],
            "like" : [
            {
                "_index" : "marvel",
                "_type" : "quotes",
                "doc" : {
                    "name": {
                        "first": "Ben",
                        "last": "Grimm"
                    },
                    "tweet": "You got no idea what I'd... what I'd give to be invisible."
                  }
            },
            {
                "_index" : "marvel",
                "_type" : "quotes",
                "_id" : "2"
            }
            ],
            "min_term_freq" : 1,
            "max_query_terms" : 12
        }
    }
}
```

### How it Works（怎么运行的）

假设我们想找到类似于给定输入文档的所有文档。 显然，输入文档本身应该是该类型的查询的最佳匹配。 而根据Lucene scoring formula，由于tf-idf最高的条件，原因主要是这样。 因此，具有最高tf-idf的输入文档的条款是该文档的良好代表，并且可以在分离式查询（或OR）中使用来检索类似的文档。 MLT查询简单地从输入文档中提取文本，分析它，通常在现场使用相同的分析器，然后选择具有最高tf-idf的顶级K项来形成这些术语的分离查询。

###### 重要

> 要在其上执行MLT的字段必须被索引并且类型为string。 另外，当使用像文档一样，_source必须被启用或字段必须存储或存储term_vector。 为了加快分析，可以帮助在索引时间存储项目向量。

例如，如果我们希望在“标题”和“tags.raw”字段上执行MLT，我们可以在索引时显式存储它们的term_vector。 我们仍然可以在“描述”和“标签”字段上执行MLT，因为默认情况下_source是启用的，但是对于这些字段的分析将不会加快速度。

```
PUT /imdb
{
    "mappings": {
        "movies": {
            "properties": {
                "title": {
                    "type": "text",
                    "term_vector": "yes"
                },
                "description": {
                    "type": "text"
                },
                "tags": {
                    "type": "text",
                    "fields" : {
                        "raw": {
                            "type" : "text",
                            "analyzer": "keyword",
                            "term_vector" : "yes"
                        }
                    }
                }
            }
        }
    }
}
```

### Parameters（参数）

唯一必需的参数就像，所有其他参数都有明智的默认值。 有三种类型的参数：一种用于指定文档输入，另一种用于术语选择和查询形成。

#### Document Input Parameters

值|描述
---|---
like|The only required parameter of the MLT query is like and follows a versatile syntax, in which the user can specify free form text and/or a single or multiple documents (see examples above). The syntax to specify documents is similar to the one used by the Multi GET API. When specifying documents, the text is fetched from fields unless overridden in each document request. The text is analyzed by the analyzer at the field, but could also be overridden. The syntax to override the analyzer at the field follows a similar syntax to the per_field_analyzer parameter of the Term Vectors API. Additionally, to provide documents not necessarily present in the index, artificial documents are also supported.
unlike|The unlike parameter is used in conjunction with like in order not to select terms found in a chosen set of documents. In other words, we could ask for documents like: "Apple", but unlike: "cake crumble tree". The syntax is the same as like.
fields|A list of fields to fetch and analyze the text from. Defaults to the _all field for free text and to all possible fields for document inputs.
like_text|The text to find documents like it.
ids or docs|A list of documents following the same syntax as the Multi GET API.

#### Term Selection Parameters

值|描述
---|---
max_query_terms|The maximum number of query terms that will be selected. Increasing this value gives greater accuracy at the expense of query execution speed. Defaults to 25.
min_term_freq|The minimum term frequency below which the terms will be ignored from the input document. Defaults to 2.
min_doc_freq|The minimum document frequency below which the terms will be ignored from the input document. Defaults to 5.
max_doc_freq|The maximum document frequency above which the terms will be ignored from the input document. This could be useful in order to ignore highly frequent words such as stop words. Defaults to unbounded (0).
min_word_length|The minimum word length below which the terms will be ignored. The old name min_word_len is deprecated. Defaults to 0.
max_word_length|The maximum word length above which the terms will be ignored. The old name max_word_len is deprecated. Defaults to unbounded (0).
stop_words|An array of stop words. Any word in this set is considered "uninteresting" and ignored. If the analyzer allows for stop words, you might want to tell MLT to explicitly ignore them, as for the purposes of document similarity it seems reasonable to assume that "a stop word is never interesting".
analyzer|The analyzer that is used to analyze the free form text. Defaults to the analyzer associated with the first field in fields.

#### Query Formation Parameters

值|描述
---|---
minimum_should_match|After the disjunctive query has been formed, this parameter controls the number of terms that must match. The syntax is the same as the minimum should match. (Defaults to "30%").
boost_terms|Each term in the formed query could be further boosted by their tf-idf score. This sets the boost factor to use when using this feature. Defaults to deactivated (0). Any other positive value activates terms boosting with the given boost factor.
include|Specifies whether the input documents should also be included in the search results returned. Defaults to false.
boost|Sets the boost value of the whole query. Defaults to 1.0.

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/5.4/query-dsl-mlt-query.html
