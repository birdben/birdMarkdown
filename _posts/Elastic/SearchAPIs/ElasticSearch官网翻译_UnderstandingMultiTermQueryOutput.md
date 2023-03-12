---
title: "ElasticSearch官网翻译_UnderstandingMultiTermQueryOutput"
date: 2017-05-28 14:34:22
tags: [Elasticsearch]
categories: [Search]
---

#### Understanding MultiTermQuery output

需要特别注意MultiTermQuery类的查询。这包括wildcards通配符，regex正则表达式和fuzzy模糊查询。这些查询发出非常冗长的响应，并且不会过于结构化。

从本质上讲，这些查询在每个分片的基础上重写自己。如果你想像通配符查询 b*，它在技术上可以匹配以字母"b"开头的所有标记。不可能枚举所有可能的组合，所以Lucene在正在评估的segment段的上下文中重写查询。例如。一个段可能包含标记[bar，baz]，因此查询重写为"bar"和"baz"的BooleanQuery组合。另一个细分可能只有标记[bakery]，所以查询重写为单一的"bakery"的TermQuery。

由于这种动态的每段重写，干净的树结构变得扭曲，不再遵循一个清晰的"谱系"，显示了一个查询如何重写到下一个。目前，我们所能做的只是道歉，并建议你折叠该查询的子项的细节，如果它太混乱。幸运的是，所有的时序统计信息都是正确的，只是不是响应中的物理布局，所以只要分析顶级的MultiTermQuery并忽略它的children就足够了，如果发现细节太难解释。

希望这将在未来的迭代中得到解决，但这是一个棘手的问题，解决并仍在进行中:)

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/2.4/_understanding_multitermquery_output.html

