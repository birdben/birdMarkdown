---
title: "ElasticSearch官网翻译_StringDatatype"
date: 2017-05-12 01:47:34
tags: [Elasticsearch]
categories: [Search]
---

#### String datatypes

字符串类型的字段接受文本值。 字符串可以细分为：

- Full text（全文本）

全文本值（如电子邮件正文）通常用于基于文本的相关性搜索，例如：找到与"quick brown fox"匹配查询最相关的文档。

这些字段是analyzed（被分析的），即通过分析器将其转换成索引之前的各个terms（词条）列表。 分析的过程允许Elasticsearch搜索每个全文本字段中的单个单词。 全文字段不用于排序，很少用于聚合（尽管significant terms aggregation是一个显着的例外）。

- Keywords（关键字）

<b>关键字是电子邮件地址，主机名，状态代码或标签等精确值。 它们通常用于过滤（查找我发布状态的所有博客文章），排序和聚合。 关键字字段是not_analyzed（不分析的）。 相反，确切的字符串值作为单个term（词条）添加到索引。</b>

以下是全文（analyzed）和关键字（not_analyzed）字符串字段的映射示例：

```
PUT my_index
{
  "mappings": {
    "my_type": {
      "properties": {
        "full_name": { (1)
          "type":  "string"
        },
        "status": {
          "type":  "string", (2)
          "index": "not_analyzed"
        }
      }
    }
  }
}
```

- (1) full_name字段是analyzed（分析的）全文字段 - index:analyzed是默认值。
- (2) status字段是not_analyzed（不分析的）关键字字段。

<b>有时，对一个字段同时使用全文（analyzed）和关键字（not_analyzed）版本是有用的：一个用于全文本搜索，另一个用于聚合和排序。 这可以通过multi-fields实现。</b>

##### 字符串字段的参数

字符串字段接受以下参数：

- analyzer : analyzer（分析器）应用于analyzed（分析的）字符串字段，无论是在索引时间还是在搜索时间（除非被search_analyzer覆盖）。默认为默认索引分析器或standard analyzer（标准分析器）。

- boost : Field-level 索引boost权重提升。 接受一个浮点数，默认为1.0。

- doc_values : 该字段是否应该以column-stride多列的方式存储在磁盘上，以便以后可以将其用于排序，聚合或脚本？ 接受true（默认）或false。

- <b>fielddata : 可以使用内存中的字段数据进行排序，聚合或脚本编写吗？接受{ "format": "disabled" } or { "format": "paged_bytes" }（默认）。未分析的字段将使用doc values优先于fielddata。</b>

- <b>fields : Multi-fields允许将相同的字符串值以多种方式索引到不同的目的，例如用于搜索的一个字段和用于排序和聚合的multi-field（多字段），或由不同分析器分析的相同字符串值。</b>

- <b>ignore_above : 不要索引或分析长于此值的任何字符串。默认为0（禁用）。</b>

- include_in_all : 字段值是否应包含在_all字段中？可以接受true或false。如果index设置为no，或者如果父对象字段将include_in_all设置为false，则默认为false。否则默认为true。

- index : 该字段是否可以被搜索？ 接收not_analyzed（默认）或者no。

- index_options : 索引中应存储哪些信息，以便搜索和高亮显示。默认为analyzed分析字段的position（位置），以及not_analyzed不分析字段的doc（文档）。

- norms : 在评分查询时是否应考虑字段长度。默认值取决于index（索引）设置：
 * analyzed字段默认为{ "enabled": true, "loading": "lazy" }。
 * not_analyzed字段默认为{ "enabled": false }.。

- null_value : 接受一个字符串值作为替换任何显式null空值的字段。默认为null，这意味着该字段被视为丢失。

- position_increment_gap : 应该在字符串数组的每个元素之间插入的fake term（伪词条）位置的数量。默认为在分析器上配置的position_increment_gap，默认为100. 100被选择，因为它防止了通过字段值匹配术语的相当大的斜率（小于100）的短语查询。

- store : 字段值是否应与_source字段分开存储和检索。 接受true或false（默认）。

- search_analyzer : analyzer（分析器）应在搜索时使用在analyzed字段。默认为analyzer设置。

- search_quote_analyzer : 在遇到短语时在搜索时使用的analyzer（分析器）。默认为search_analyzer设置。

- similarity : 应该使用哪种评分算法或相似度。默认为default，它使用TF / IDF。

- term_vector : 是否应为analyzed（分析的）字段存储term vectors（词条向量）。默认为no。


参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/2.4/string.html
