---
title: "ElasticSearch官网翻译_SimpleQueryStringQuery"
date: 2017-06-18 17:04:39
tags: [Elasticsearch]
categories: [Search]
---

## Simple Query String Query

一个使用SimpleQueryParser解析其上下文的查询。 与常规query_string查询不同，simple_query_string查询永远不会抛出异常，并丢弃查询的无效部分。 这是一个例子：

```
GET /_search
{
  "query": {
    "simple_query_string" : {
        "query": "\"fried eggs\" +(eggplant | potato) -frittata",
        "analyzer": "snowball",
        "fields": ["body^5","_all"],
        "default_operator": "and"
    }
  }
}
```

simple_query_string顶级参数包括：

参数|描述
---|---
query|要解析的实际查询。 请参阅下面的语法。
fields|执行解析查询的字段。 默认为index.query.default_field索引设置，默认为_all。
default_operator|如果未指定显式运算符，则使用默认运算符。 例如，使用OR的默认运算符，"capital of Hungary"将转换为"capital OR of OR Hungary"，并且使用AND的默认运算符，将相同的查询转换为"capital AND of AND Hungary"。 默认值为OR。
analyzer|分析器用于分析查询的每个词条，当创建复合查询时。
flags|指定要启用的simple_query_string的哪些功能。 默认为ALL。
analyze_wildcard|是否应自动分析前缀查询的词条。 如果是true，将尽力分析前缀。 但是，一些分析器将无法根据词条的前缀提供有意义的结果。 默认为false。
lenient|如果设置为true，将会导致基于格式的失败（例如提供文本给数字类型的字段）被忽略。
minimum_should_match|要返回的文档必须匹配的最小子句数。 有关选项的完整列表，请参阅minimum_should_match文档。
quote_field_suffix|附加到查询字符串引用部分的字段的后缀。 这允许使用具有不同分析链的字段来进行精确匹配。 看这里一个全面的例子。
all_fields|对可以查询的映射中检测到的所有字段执行查询。 默认情况下将使用_all字段禁用，并且在索引设置没有指定default_field，并且没有明确指定fields。

#### Simple Query String Syntax

simple_query_string支持以下特殊字符：

- + 表示AND操作
- | 表示OR操作
- - 否定单个词元
- " 包装了一些词元来表示一个搜索短语
- * 在词条尾部表示前缀查询
- ( 和 ) 表示优先
- ~N 一个字后面的N表示编辑距离（fuzziness）
- ~N 短语中的〜N表示slop

为了搜索这些特殊字符，它们将需要用\进行转义。

请注意，此语法可能具有不同的行为，具体取决于default_operator值。 例如，考虑以下查询：

```
GET /_search
{
    "query": {
        "simple_query_string" : {
            "fields" : ["content"],
            "query" : "foo bar -baz"
        }
    }
}
```

你可能希望只返回只包含"foo"或"bar"的文档，只要它们不包含"baz"，但是由于default_operator为OR，这意味着匹配的文档包含"foo"或包含"bar"或者不包含"baz"，如果这不是预期的结果，则查询可以切换到不会返回包含"baz"的文档的"foo bar +-baz"。

#### Default Field（默认字段）

当在查询字符串语法中未明确指定要搜索的字段时，将使用index.query.default_field来导出要搜索的字段。 它默认为_all字段。

如果_all字段被禁用，并且在请求中没有指定fields，则simple_query_string查询将自动尝试确定索引映射中可查询的现有字段，并对这些字段执行搜索。

#### Multi Field（多字段）

fields参数还可以包括基于模式的字段名称，允许自动扩展到相关字段（包括动态引入的字段）。 例如：

```
GET /_search
{
    "query": {
        "simple_query_string" : {
            "fields" : ["content", "name.*^5"],
            "query" : "foo bar baz"
        }
    }
}
```

#### Flags（标识）

simple_query_string支持多个标识来指定应该启用哪些解析功能。 它可以指定flags参数为|分割的字符串：

```
GET /_search
{
    "query": {
        "simple_query_string" : {
            "query" : "foo | bar + baz*",
            "flags" : "OR|AND|PREFIX"
        }
    }
}
```

可用的标识是：ALL，NONE，AND，OR，NOT，PREFIX，PHRASE，PRECEDENCE，ESCAPE，WHITESPACE，FUZZY，NEAR和SLOP。

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/5.4/query-dsl-simple-query-string-query.html
