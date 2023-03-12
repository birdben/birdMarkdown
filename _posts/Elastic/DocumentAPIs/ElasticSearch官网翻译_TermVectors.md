---
title: "ElasticSearch官网翻译_TermVectors"
date: 2017-06-01 18:17:38
tags: [Elasticsearch]
categories: [Search]
---

## Term Vectors（词条向量）

返回有关特定文档字段中的词条的信息和统计信息。 文档可以存储在索引中或由用户人工提供。 Term vectors（词条向量）默认为实时，不是准实时。 这可以通过将realtime参数设置为false来更改。

```
GET /twitter/tweet/1/_termvectors
```

或者，你可以使用url中的参数指定取回字段的信息

```
GET /twitter/tweet/1/_termvectors?fields=message
```

或通过在请求主体中添加请求的字段（参见下面的示例）。 也可以使用通配符指定字段，类似于multi match query（多匹配查询）

###### 警告

> 请注意，/_termvector的使用在2.0中已被弃用，并由/_termvectors替代。

### Return values（返回值）

可以请求三种类型的值：term information（词条信息），term statistics（词条统计）和field statistics（字段统计）。 默认情况下，所有字段都返回所有词条信息和字段统计信息，但不包含词条统计信息。

#### Term information（词条信息）

- term_freq : 字段的词条频率（总是返回）
- position : 词条位置（positions : true）
- start_offset, end_offset : 开始和结束偏移（offsets : true）
- 词条的有效装载量（payloads : true），以base64编码的字节

如果请求的信息没有存储在索引中，如果可能，它将被即时计算。 另外，term vectors甚至也可以被计算不存在于索引中，但由用户提供的文档。

###### 警告

> 开始和结束偏移假设正在使用UTF-16编码。 如果要使用这些offset偏移量以获取产生此标识的原始文本，则应确保使用UTF-16对正在使用子字符串的字符串进行编码。

#### Term statistics（词条统计）

将term_statistics设置为true（默认为false）将返回

- ttf : 总词条频率（所有文件中的词条频率）
- doc_freq : 文档频率（包含当前词条的文档数）

默认情况下，这些值不会返回，因为词条统计可能会对性能产生严重影响。

#### Field statistics（字段统计）

将field_statistics设置为false（默认值为true）将省略：

- doc_count : 文档数（包含此字段的文档数）
- sum_doc_freq : 文档频率的总和（该字段中所有词条的文档频率的总和）
- sum_ttf : 总词条频率的总和（该字段中每个词条的总词条频率的总和）

#### Terms Filtering（词条过滤器）

使用参数filter，返回的词条也可以根据其tf-idf分数进行过滤。 这可能是有用的，以便找到文档的良好特征向量。 此功能的工作方式与"More Like This Query"的第二阶段相似。 参见示例5的使用。

支持以下子参数：

值|描述
---|---
max_num_terms|每个字段必须返回的最大词条数。 默认为25。
min_term_freq|在源文档中忽略少于此频率的单词。 默认为1。
max_term_freq|在源文档中忽略超过此频率的单词。 默认为unbounded（无界）。
min_doc_freq|忽略至少在这么多文档中都没有出现的词条。 默认为1。
max_doc_freq|忽略发生在多个文档中的单词。 默认为unbounded（无界）。
min_word_length|低于单词的最小字长将被忽略。 默认为0。
max_word_length|超过单词的最大字长将被忽略。 默认为无界（0）。

### Behaviour（行为）

词条和字段统计数据不准确。 删除的文档不被考虑。 这些信息只能用于所请求文档所在的分片。因此，词条和字段统计信息仅用作相对度量，而绝对数字在此上下文中无意义。 默认情况下，当请求人造文档的词条向量时，随机选择获取统计信息的分片。 仅使用路由命中特定的分片。

#### Example: Returning stored term vectors（例子：返回存储的词条向量）

首先，我们创建一个存储词条向量，有效装载量等的索引：

```
PUT /twitter/
{ "mappings": {
    "tweet": {
      "properties": {
        "text": {
          "type": "text",
          "term_vector": "with_positions_offsets_payloads",
          "store" : true,
          "analyzer" : "fulltext_analyzer"
         },
         "fullname": {
          "type": "text",
          "term_vector": "with_positions_offsets_payloads",
          "analyzer" : "fulltext_analyzer"
        }
      }
    }
  },
  "settings" : {
    "index" : {
      "number_of_shards" : 1,
      "number_of_replicas" : 0
    },
    "analysis": {
      "analyzer": {
        "fulltext_analyzer": {
          "type": "custom",
          "tokenizer": "whitespace",
          "filter": [
            "lowercase",
            "type_as_payload"
          ]
        }
      }
    }
  }
}
```

然后，我们添加一些文档：

```
PUT /twitter/tweet/1
{
  "fullname" : "John Doe",
  "text" : "twitter test test test "
}

PUT /twitter/tweet/2
{
  "fullname" : "Jane Doe",
  "text" : "Another twitter test ..."
}
```

以下请求返回文档1（John Doe）中text字段的所有信息和统计信息：

```
GET /twitter/tweet/1/_termvectors
{
  "fields" : ["text"],
  "offsets" : true,
  "payloads" : true,
  "positions" : true,
  "term_statistics" : true,
  "field_statistics" : true
}
```

响应：

```
{
    "_id": "1",
    "_index": "twitter",
    "_type": "tweet",
    "_version": 1,
    "found": true,
    "took": 6,
    "term_vectors": {
        "text": {
            "field_statistics": {
                "doc_count": 2,
                "sum_doc_freq": 6,
                "sum_ttf": 8
            },
            "terms": {
                "test": {
                    "doc_freq": 2,
                    "term_freq": 3,
                    "tokens": [
                        {
                            "end_offset": 12,
                            "payload": "d29yZA==",
                            "position": 1,
                            "start_offset": 8
                        },
                        {
                            "end_offset": 17,
                            "payload": "d29yZA==",
                            "position": 2,
                            "start_offset": 13
                        },
                        {
                            "end_offset": 22,
                            "payload": "d29yZA==",
                            "position": 3,
                            "start_offset": 18
                        }
                    ],
                    "ttf": 4
                },
                "twitter": {
                    "doc_freq": 2,
                    "term_freq": 1,
                    "tokens": [
                        {
                            "end_offset": 7,
                            "payload": "d29yZA==",
                            "position": 0,
                            "start_offset": 0
                        }
                    ],
                    "ttf": 2
                }
            }
        }
    }
}
```

#### Example: Generating term vectors on the fly（例子：自动生成词条向量）

未明确存储在索引中的词条向量将自动计算。 以下请求返回文档1中字段的所有信息和统计信息，即使词条尚未明确存储在索引中。 请注意，对于text字段，词条不会重新生成。

```
GET /twitter/tweet/1/_termvectors
{
  "fields" : ["text", "some_field_without_term_vectors"],
  "offsets" : true,
  "positions" : true,
  "term_statistics" : true,
  "field_statistics" : true
}
```

#### Example: Artificial documents（人造文档）

还可以为人造文档生成词条向量，也就是生成索引中不存在的文档。 例如，以下请求将返回与示例1中相同的结果。所使用的映射由index和type确定。

如果动态映射打开（默认），则不在原始映射中的文档字段将被动态创建。

```
GET /twitter/tweet/_termvectors
{
  "doc" : {
    "fullname" : "John Doe",
    "text" : "twitter test test test"
  }
}
```

#### Per-field analyzer（为每个字段设置分析器）

另外，可以通过使用per_field_analyzer参数来提供不同于当前的分析器。 这对于以任何方式生成词条向量是有用的，特别是在使用人造文档时。 当为已经存储词条向量的字段供分析器时，将重新生成词条向量。

```
GET /twitter/tweet/_termvectors
{
  "doc" : {
    "fullname" : "John Doe",
    "text" : "twitter test test test"
  },
  "fields": ["fullname"],
  "per_field_analyzer" : {
    "fullname": "keyword"
  }
}
```

响应：

```
{
  "_index": "twitter",
  "_type": "tweet",
  "_version": 0,
  "found": true,
  "took": 6,
  "term_vectors": {
    "fullname": {
       "field_statistics": {
          "sum_doc_freq": 2,
          "doc_count": 4,
          "sum_ttf": 4
       },
       "terms": {
          "John Doe": {
             "term_freq": 1,
             "tokens": [
                {
                   "position": 0,
                   "start_offset": 0,
                   "end_offset": 8
                }
             ]
          }
       }
    }
  }
}
```

#### Example: Terms filtering（例子：词条过滤器）

最后，返回的词条可以根据他们的tf-idf分数进行过滤。 在下面的例子中，我们从人造文档中给定的"plot"字段值中获取三个最"感兴趣"的关键字。 请注意，关键字"Tony"或任何stop words停止词不是响应的一部分，因为它们的tf-idf必须太低。

```
GET /imdb/movies/_termvectors
{
    "doc": {
      "plot": "When wealthy industrialist Tony Stark is forced to build an armored suit after a life-threatening incident, he ultimately decides to use its technology to fight against evil."
    },
    "term_statistics" : true,
    "field_statistics" : true,
    "positions": false,
    "offsets": false,
    "filter" : {
      "max_num_terms" : 3,
      "min_term_freq" : 1,
      "min_doc_freq" : 1
    }
}
```

响应：

```
{
   "_index": "imdb",
   "_type": "movies",
   "_version": 0,
   "found": true,
   "term_vectors": {
      "plot": {
         "field_statistics": {
            "sum_doc_freq": 3384269,
            "doc_count": 176214,
            "sum_ttf": 3753460
         },
         "terms": {
            "armored": {
               "doc_freq": 27,
               "ttf": 27,
               "term_freq": 1,
               "score": 9.74725
            },
            "industrialist": {
               "doc_freq": 88,
               "ttf": 88,
               "term_freq": 1,
               "score": 8.590818
            },
            "stark": {
               "doc_freq": 44,
               "ttf": 47,
               "term_freq": 1,
               "score": 9.272792
            }
         }
      }
   }
}
```

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/5.4/docs-termvectors.html
