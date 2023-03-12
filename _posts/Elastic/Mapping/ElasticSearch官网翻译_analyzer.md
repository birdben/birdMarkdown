---
title: "ElasticSearch官网翻译_analyzer"
date: 2017-05-13 12:11:25
tags: [Elasticsearch]
categories: [Search]
---

#### analyzer（分析器）

<b>
analyzed（被分析）的字符串字段的值通过analyzer（分析器）传递，以将字符串转换为tokens（词元）或terms（词条）。例如，字符串"The quick Brown Foxes."。根据使用哪种analyzer（分析器），可以被解析为tokens（词元）：quick, brown, fox。这些是为该字段索引的实际terms（词条），这使得有​​效地搜索文本大块内的单个单词。
</b>

该分析过程不仅发生在索引时，而且在查询时也需要：查询字符串需要通过相同（或类似的）分析器传递，以便它尝试查找与那些存在于索引中同样格式的词条。

<b>
Elasticsearch配备了许多pre-defined analyzers（预定义的分析器），可以在不进一步配置的情况下使用。它还附带许多character filters（字符过滤器），tokenizers（分词器）和, Token Filters（标记过滤器），可以组合以配置每个索引的自定义分析器。

每个query（查询），每个field（字段）或每个index（索引）可以指定分析器。在索引时间，Elasticsearch将按以下顺序查找分析器：

- 在field mapping（字段映射）中定义的analyzer分析器。
- 在index setting（索引设置）中名为default的analyzer分析器。
- standard标准分析器。

在查询时，还有几层：

- 在full-text query（全文查询）中定义的analyzer分析器。
- 在field mapping（字段映射）中定义的search_analyzer分析器。
- 在field mapping（字段映射）中定义的analyzer分析器。
- 在index setting（索引设置）中名为default_search的analyzer分析器。
- 在index setting（索引设置）中名为default的analyzer分析器。
- standard标准分析器。
</b>

为特定字段指定分析器的最简单方法是在字段映射中进行定义，如下所示：

```
PUT my_index
{
  "mappings": {
    "my_type": {
      "properties": {
        "text": { (1)
          "type": "string",
          "fields": {
            "english": { (2)
              "type":     "string",
              "analyzer": "english"
            }
          }
        }
      }
    }
  }
}

GET my_index/_analyze?field=text (3)
{
  "text": "The quick Brown Foxes."
}

GET my_index/_analyze?field=text.english (4)
{
  "text": "The quick Brown Foxes."
}
```

- (1) text字段使用默认standard标准分析器。
- (2) text.english multi-field（多字段）使用english英文分析器，可以删除stop words（停止词）并应用词干。
- (3) 这返回tokens（词元）： [ the, quick, brown, foxes ]。
- (4) 这会返回tokens（词元）： [ quick, brown, fox ]。

##### search_quote_analyzer（短语查询分析器）

<b>
search_quote_analyzer设置允许你为phrases（短语）指定分析器，这在处理phrase queries（短语查询）的禁用stop words（停止词）时特别有用。

要使用三个分析器设置来禁用短语的stop words（停止词）：

- 用于索引所有术语（包括停止词）的分析器设置
- 用于非词组查询的search_analyzer设置，将删除停止词
- 用于短语查询的search_quote_analyzer设置，不会删除停止词
</b>

```
PUT /my_index
{
   "settings":{
      "analysis":{
         "analyzer":{
            "my_analyzer":{ (1)
               "type":"custom",
               "tokenizer":"standard",
               "filter":[
                  "lowercase"
               ]
            },
            "my_stop_analyzer":{ (2)
               "type":"custom",
               "tokenizer":"standard",
               "filter":[
                  "lowercase",
                  "english_stop"
               ]
            }
         },
         "filter":{
            "english_stop":{
               "type":"stop",
               "stopwords":"_english_"
            }
         }
      }
   },
   "mappings":{
      "my_type":{
         "properties":{
            "title": {
               "type":"string",
               "analyzer":"my_analyzer", (3)
               "search_analyzer":"my_stop_analyzer", (4)
               "search_quote_analyzer":"my_analyzer" (5)
              }
            }
         }
      }
   }
}
```

```
PUT my_index/my_type/1
{
   "title":"The Quick Brown Fox"
}

PUT my_index/my_type/2
{
   "title":"A Quick Brown Fox"
}

GET my_index/my_type/_search
{
   "query":{
      "query_string":{
         "query":"\"the quick brown fox\"" (1)
      }
   }
}
```

<b>

- (1) my_analyzer分析器定义tokens（词元）所有terms（词条）都包括stop words（停止词）
- (2) my_stop_analyzer分析器定义可以删除stop words（停止词）
- (3) analyzer设置指定将在索引时使用的my_analyzer分析器
- (4) search_analyzer设置指定使用的my_stop_analyzer分析器，并删除非短语查询的stop words（停止词）
- (5) search_quote_analyzer设置指定使用的my_analyzer分析器，并确保stop words（停止词）不会从短语查询中删除
- (1) 由于查询是用引号括起来的，因此它被检测为短语查询，因此search_quote_analyzer会启动并确保停止词不会从查询中删除。 my_analyzer分析器将返回以下 [the, quick, brown, fox] 与文档匹配的tokens（词元）。同时，将通过my_stop_analyzer分析器分析term精确查询，该分析器将过滤掉停止词。因此，搜索"The quick brown fox"或者"A quick brown fox"将返回两份文档，因为这两个文档都包含以下词元[quick, brown, fox]。没有search_quote_analyzer，将不可能对短语查询进行完全匹配，因为短语查询的停止词将被删除，从而导致两个文档匹配。

</b>

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/2.4/analyzer.html
