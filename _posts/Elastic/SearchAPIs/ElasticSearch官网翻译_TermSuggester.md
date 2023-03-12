---
title: "ElasticSearch官网翻译_TermSuggester"
date: 2017-05-27 11:59:09
tags: [Elasticsearch]
categories: [Search]
---

#### Term suggester

###### 注意

为了了解建议的格式，请先阅读Suggesters页面。

Term suggester建议词条基于编辑距离的。 提供的建议文本是analyzed分析的在提出词条被建议之前。 每个分析的建议文字标记提供了建议的词条。 Term suggester不会将查询视为请求的一部分。

##### Common suggest options

值|描述
---|---
text|建议文本。 建议文本是需要在全局或每个建议中设置的必需选项。
field|从中获取候选建议的字段。 这是需要在全局或每个建议中设置的必需选项。
analyzer|分析器用于分析建议文本。 默认为建议字段的search analyzer搜索分析器。
size|每个建议文本标记的最大返回值。
sort|定义如何按建议文本词条排序建议。 两个可能的值： 
 |- score：首先按分数排序，然后按文档频率排序，然后是词条本身。 
 |- frequency：首先按文档频率排序，然后按照相似度得分，然后是词条本身。
suggest_mode|建议模式控制什么建议被包括或控制什么建议文本词条，应提出建议。可以指定三个可能的值： 
 |- missing：仅提供不在索引中的建议文本词条的建议。 这是默认值。 
 |- popular：只建议发生在更多的文档，然后原始的建议文本词条。 
 |- always：根据建议文本中的词条建议任何匹配的建议。

##### Other term suggest options

值|描述
---|---
lowercase_terms|在文本分析后小写化建议文本词条。
max_edits|最大编辑距离候选人建议可以被认为是一个建议。 只能是1到2之间的值。任何其他值都会导致抛出请求错误。 默认为2。
prefix_length|必须匹配的最小前缀字符数是候选建议。 默认为1。 增加此数字会提高拼写检查的性能。 通常拼写错误不会在词条的开头出现。 （旧名称“prefix_len”已弃用）
min_word_length|建议文本词条必须包含的最小长度。 默认为4.（旧名称“min_word_len”已弃用）
shard_size|设置从每个分片中检索的最大建议数。 在合并阶段期间，只有根据size选项返回前N个建议。 默认为size选项。 将其设置为高于size的值可能有用，以便以性能为代价获得更准确的文档频率进行拼写更正。 由于词条被划分到分片，因此拼写校正的分片级别文档频率可能不准确。 增加这些将使这些文档的频率更加精确。
max_inspections|用于与shards_size相乘的一个因子，以便在分片级别上检查更多的候选拼写更正。 可以以性能为代价来提高准确性。 默认为5。
min_doc_freq|一个建议的文档数量的最小阈值应该出现在这里。这可以被指定为一个绝对数字或相对于文档数量的相对百分比。 这可以通过仅提出高频率来提高质量。 默认为0f，未启用。 如果指定了高于1的值，则该数字不能为小数。 此选项使用分片级文档频率。
max_term_freq|建议文本标记的文档数量的最大阀值可以存在，以便包括在内。 可以是相对百分比数（例如0.4）或表示文档频率的绝对数。 如果指定了高于1的值，则无法指定小数。 默认为0.01f。 这可以用于排除高频率词条的拼写检查。 高频词条通常拼写正确，这也提高了拼写检查的性能。 此选项使用分片级文档频率。
string_distance|哪个字符串距离实现用于比较类似的建议词条。 可以指定五个可能的值：
 |internal - 基于damerau_levenshtein的默认值，但对于比较索引中词条的字符串距离进行高度优化。 
 |Damerau_levenshtein - 基于Damerau-Levenshtein算法的字符串距离算法。 
 |Levenstein - 基于Levenstein编辑距离算法的字符串距离算法。 
 |jarowinkler - 基于Jaro-Winkler算法的字符串距离算法。 
 |ngram - 基于字符n-gram的字符串距离算法。

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/2.4/search-suggesters-term.html

