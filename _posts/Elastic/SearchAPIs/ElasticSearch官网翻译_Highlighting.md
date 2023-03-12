---
title: "ElasticSearch官网翻译_Highlighting"
date: 2017-05-25 01:05:41
tags: [Elasticsearch]
categories: [Search]
---

#### Highlighting

<b>允许在一个或多个字段上高亮显示搜索结果。 实现使用了lucene的highlighter，fast-vector-highlighter或postings-highlighter。 </b>以下是搜索请求正文的示例：

```
{
    "query" : {...},
    "highlight" : {
        "fields" : {
            "content" : {}
        }
    }
}
```

在上述情况下，每个搜索命中将高亮显示content字段（每个搜索命中中将会有另一个元素，称为highlight（高亮），其中包括高亮显示的字段和高亮显示的片段）。

###### 注意

<b>
为了执行高亮显示，需要字段中的实际内容。 如果字段考虑设置为存储（映射中存储设置为true），则将使用它，否则将加载实际的_source，并从中提取相关字段。

_all字段无法从_source提取，因此只能将其映射为将store存储设置为true，才能用于高亮显示。

字段名称支持通配符符号。 例如，使用comment_ *将导致所有与表达式匹配的字符串字段被高亮显示。 请注意，所有其他字段不会突出显示。 如果你使用自定义的映射，并且无论如何要在字段上高亮显示，则必须显式提供字段名称。
</b>

##### Plain highlighter（普通高亮）

<b>
默认选择的高亮类型是plain（普通的），并使用Lucene highlighter。 在phrase queries（短语查询）中理解单词重要性和任何单词定位标准方面，都很难反映查询匹配逻辑。
</b>

###### 警告

如果你想要在复杂查询的大量文档中高亮显示许多字段，则此highlighter将不会很快。 在努力准确反映查询逻辑中，它创建了一个微小的内存索引，并通过Lucene的查询执行计划程序重新运行原始查询条件，以获取当前文档的低级匹配信息。 对于需要高亮显示的每个字段和每个文档，都会重复此操作。 如果这会在你的系统中出现性能问题，请考虑使用其他highlighter。

##### Postings highlighter（帖子高亮）

<b>
如果在映射中将index_options设置为offsets，则将使用postings highlighter，而不是plain highlighter。 postings highlighter：

- 更快，因为它不需要重新分析要高亮显示的文本：文档越大，性能增益就越好
- 需要比term_vectors少的磁盘空间，需要fast vector highlighter
- 将文本分解成句子并高亮显示。 非常适合自然语言，而不是与包含例如html标记的字段
- 将文档视为整个语料库，并使用BM25算法对单个句子进行评分，如同它们是该语料库中的文档
</b>

以下是设置content字段以允许在其上使用postings highlighter来高亮显示的示例：

```
{
    "type_name" : {
        "content" : {"index_options" : "offsets"}
    }
}
```

###### 注意

<b>
请注意，postings highlighter旨在执行简单的查询词条高亮显示，无论其位置如何。 这意味着当用于例如与短语查询结合使用时，它将高亮显示查询所构成的所有terms词条，而不管它们实际上是查询匹配的一部分，有效地忽略了它们的位置。
</b>

###### 警告

<b>
postings highlighter不支持高亮显示一些复杂的查询，例如类型设置为match_phrase_prefix的match匹配查询。 在这种情况下，不会返回高亮显示的片段。
</b>

##### Fast vector highlighter（快速矢量高亮）

<b>
如果通过在映射中将term_vector设置为with_positions_offsets来提供term_vector信息，则将使用fast vector highlighter，而不是plain highlighter。 fast vector highlighter：

- 特别适用于大字段（> 1MB）
- 可以使用boundary_chars，boundary_max_scan和fragment_offset进行定制（见下文）
- 需要将term_vector设置为with_positions_offsets，这会增加索引的大小
- 可以将来自多个字段的匹配组合成一个结果。 请参阅matched_fields
- 可以在不同的位置分配不同的权重以匹配，例如，当高亮显示提升词组匹配超过期限匹配的Boosting Query时，比如短语匹配被排除在term词条匹配之上
</b>

这是一个设置content字段的示例，在其上使用fast vector highlighter来高亮显示（这将导致索引更大）：

```
{
    "type_name" : {
        "content" : {"term_vector" : "with_positions_offsets"}
    }
}
```

##### Force highlighter type

<b>type字段允许强制指定highlighter type。 这对于在需要启用term_vectors的字段上需要使用plain highlighter时非常有用。 允许的值为：plain，postings和fvh。 以下是强制使用plain highlighter的示例：</b>

```
{
    "query" : {...},
    "highlight" : {
        "fields" : {
            "content" : {"type" : "plain"}
        }
    }
}
```

##### Force highlighting on source

强制高亮显示基于source的高亮字段，即使字段是单独存储的。 默认为false。

```
{
    "query" : {...},
    "highlight" : {
        "fields" : {
            "content" : {"force_source" : true}
        }
    }
}
```

##### Highlighting Tags

默认情况下，高亮显示将使用<em>和</em>包裹住高亮显示的文本。 这可以通过设置pre_tags和post_tags来控制，例如：

```
{
    "query" : {...},
    "highlight" : {
        "pre_tags" : ["<tag1>"],
        "post_tags" : ["</tag1>"],
        "fields" : {
            "_all" : {}
        }
    }
}
```

使用fast vector highlighter可以有更多的标签，并且按照“重要性”被排序。

```
{
    "query" : {...},
    "highlight" : {
        "pre_tags" : ["<tag1>", "<tag2>"],
        "post_tags" : ["</tag1>", "</tag2>"],
        "fields" : {
            "_all" : {}
        }
    }
}
```

还有内置“tag”模式，目前有一个单一的模式，称为styled，具有以下pre_tags：

```
<em class="hlt1">, <em class="hlt2">, <em class="hlt3">,
<em class="hlt4">, <em class="hlt5">, <em class="hlt6">,
<em class="hlt7">, <em class="hlt8">, <em class="hlt9">,
<em class="hlt10">
```

和</em>作为post_tags。 如果你认为有更好的内置标签模式，只需发送电子邮件到邮件列表或提出问题。 以下是切换标记模式的示例：

```
{
    "query" : {...},
    "highlight" : {
        "tags_schema" : "styled",
        "fields" : {
            "content" : {}
        }
    }
}
```

##### Encoder

编码器参数可用于定义高亮显示的文本将如何编码。 它可以是default（无编码）或html（将转义html，如果你使用html highlighting tags）。

##### Highlighted Fragments（高亮片段）

高亮显示的每个字段可以以字符（默认为100）和最大返回片段数（默认为5）来控制高亮显示的片段的大小。 例如：

```
{
    "query" : {...},
    "highlight" : {
        "fields" : {
            "content" : {"fragment_size" : 150, "number_of_fragments" : 3}
        }
    }
}
```

<b>使用postings highlighter时，fragment_size将被忽略，因为它会输出句子，而不管其长度如何。</b>

除此之外，可以指定高亮显示的片段需要按分数排序：

```
{
    "query" : {...},
    "highlight" : {
        "order" : "score",
        "fields" : {
            "content" : {"fragment_size" : 150, "number_of_fragments" : 3}
        }
    }
}
```

如果number_of_fragments值设置为0，则不会生成片段，而是返回该字段的整个内容，当然也会高亮显示。 如果需要高亮显示短文本（如文档标题或地址），但不需要分片，这可能非常方便。 请注意，在这种情况下，fragment_size将被忽略。

```
{
    "query" : {...},
    "highlight" : {
        "fields" : {
            "_all" : {},
            "bio.title" : {"number_of_fragments" : 0}
        }
    }
}
```

<b>
当使用fast-vector-highlighter时，可以使用fragment_offset参数来控制开始高亮显示的边距。

在没有匹配片段高亮显示的情况下，默认是不返回任何东西。 相反，我们可以通过将no_match_size（默认为0）设置为要返回的文本的长度，从字段开始返回一段文本。 实际长度可能会短于指定的字符，因为它试图在字边界上断开。 当使用postings highlighter时，无法控制片段的实际大小，因此只要no_match_size大于0，第一个句子就会返回。
</b>

```
{
    "query" : {...},
    "highlight" : {
        "fields" : {
            "content" : {
                "fragment_size" : 150,
                "number_of_fragments" : 3,
                "no_match_size": 150
            }
        }
    }
}
```

##### Highlight query（高亮查询）

<b>
还可以通过设置highlight_query高亮显示搜索查询以外的查询。 如果你使用rescore查询，这是非常有用的，因为默认情况下高亮显示不会考虑这些。 Elasticsearch不会验证highlight_query以任何方式包含搜索查询，因此可以定义它，所以合法的查询结果根本不会高亮显示。 一般来说，最好是在highlight_query中包含搜索查询。 以下是在highlight_query中包含搜索查询和rescore查询的示例。
</b>

```
{
    "fields": [ "_id" ],
    "query" : {
        "match": {
            "content": {
                "query": "foo bar"
            }
        }
    },
    "rescore": {
        "window_size": 50,
        "query": {
            "rescore_query" : {
                "match_phrase": {
                    "content": {
                        "query": "foo bar",
                        "phrase_slop": 1
                    }
                }
            },
            "rescore_query_weight" : 10
        }
    },
    "highlight" : {
        "order" : "score",
        "fields" : {
            "content" : {
                "fragment_size" : 150,
                "number_of_fragments" : 3,
                "highlight_query": {
                    "bool": {
                        "must": {
                            "match": {
                                "content": {
                                    "query": "foo bar"
                                }
                            }
                        },
                        "should": {
                            "match_phrase": {
                                "content": {
                                    "query": "foo bar",
                                    "phrase_slop": 1,
                                    "boost": 10.0
                                }
                            }
                        },
                        "minimum_should_match": 0
                    }
                }
            }
        }
    }
}
```

请注意，在这种情况下，文本片段的得分由Lucene高亮框架计算。 有关实现细节，您可以检查ScoreOrderFragmentsBuilder.java类。 另一方面，当使用postings highlighter时，如上所述使用BM25算法对片段进行评分。

##### Global Settings（全局设置）

高亮显示的设置可以在全局级别设置，然后在字段级别被覆盖。

```
{
    "query" : {...},
    "highlight" : {
        "number_of_fragments" : 3,
        "fragment_size" : 150,
        "tag_schema" : "styled",
        "fields" : {
            "_all" : { "pre_tags" : ["<em>"], "post_tags" : ["</em>"] },
            "bio.title" : { "number_of_fragments" : 0 },
            "bio.author" : { "number_of_fragments" : 0 },
            "bio.content" : { "number_of_fragments" : 5, "order" : "score" }
        }
    }
}
```

##### Require Field Match（必填字段匹配）

require_field_match可以被设置为false，这将导致任何字段被高亮显示，无论查询是否具体匹配。 默认行为为true，这意味着只有保持查询匹配的字段才会被高亮显示。

```
{
    "query" : {...},
    "highlight" : {
        "require_field_match": false
        "fields" : {...}
    }
}
```

##### Boundary Characters（边界符号）

当使用fast vector highlighter高亮显示一个字段时，可以配置boundary_chars定义什么构成高亮显示的边界。 它是一个单个字符串，其中定义了每个边界字符。 它默认为.,!? \t\n.。

boundary_max_scan允许控制查找边界字符的距离，默认为20。

##### Matched Fields（匹配字段）

Fast Vector Highlighter可以组合多个字段上的匹配，以使用matched_fields高亮显示单个字段。 对于以不同方式分析相同字符串的多字段，这是最直观的。 所有matched_fields必须将term_vector设置为with_positions_offsets，但只加入匹配组合的字段，因此只有该字段才能将store存储设置为yes。

在下面的例子中，content使用english analyzer英文分析器进行分析，content.plain使用standard analyzer标准分析器进行分析。

```
{
    "query": {
        "query_string": {
            "query": "content.plain:running scissors",
            "fields": ["content"]
        }
    },
    "highlight": {
        "order": "score",
        "fields": {
            "content": {
                "matched_fields": ["content", "content.plain"],
                "type" : "fvh"
            }
        }
    }
}
```

以上均符合"run with scissors"和"running with scissors"，并将高亮显示"running"和"scissors"，而不是"run"。 如果两个短语都出现在一个大的文档中，那么在片段列表中"running with scissors"的顺序在"run with scissors"之上，因为在该片段中有更多的匹配。

```
{
    "query": {
        "query_string": {
            "query": "running scissors",
            "fields": ["content", "content.plain^10"]
        }
    },
    "highlight": {
        "order": "score",
        "fields": {
            "content": {
                "matched_fields": ["content", "content.plain"],
                "type" : "fvh"
            }
        }
    }
}
```

上述"run"以及"running"和"scissors"，但是由于明显匹配（"running"）被提升，所以"running with scissors"排序仍然高于"run with scissors"。

```
{
    "query": {
        "query_string": {
            "query": "running scissors",
            "fields": ["content", "content.plain^10"]
        }
    },
    "highlight": {
        "order": "score",
        "fields": {
            "content": {
                "matched_fields": ["content.plain"],
                "type" : "fvh"
            }
        }
    }
}
```

上述查询不会高亮显示"run"或"scissor"，但显示没有列出在匹配字段中组合匹配的字段（内容）。

###### 注意

在技术上，将匹配的字段添加到match_fields中，并且不会与匹配组合的字段共享相同的底层字符串。 结果可能没有什么意义，如果其中一个匹配关闭文本的末尾，则整个查询将失败。

##### 注意

将matched_fields设置为非空数组时所需的开销很少，因此始终优选

```
    "highlight": {
        "fields": {
            "content": {}
        }
    }
```

较于

```
    "highlight": {
        "fields": {
            "content": {
                "matched_fields": ["content"],
                "type" : "fvh"
            }
        }
    }
```

##### Phrase Limit

fast-vector-highlighter有一个phrase_limit参数，可以防止它分析太多的短语和吃大量的内存。 它默认为256，所以只有文档中前256个匹配的短语被考虑。 你可以使用phrase_limit参数提高限制，但请记住，打分更多短语会消耗更多的时间和内存。

如果使用matched_fields，请记住，每个匹配字段的phrase_limit短语都被考虑。

##### Field Highlight Order

Elasticsearch按照发送的顺序高亮显示字段。 每个json spec对象是无序的，但是如果你需要明确指出字段被高亮显示的顺序，那么你可以使用这样的fields的数组：

```
    "highlight": {
        "fields": [
            {"title":{ /*params*/ }},
            {"text":{ /*params*/ }}
        ]
    }
```

Elasticsearch内置的highlighters都没有关注字段高亮显示的顺序，但插件有可能。

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/2.4/search-request-highlighting.html

