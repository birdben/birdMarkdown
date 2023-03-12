---
title: "ElasticSearch官网翻译_QuerySection"
date: 2017-05-28 13:07:59
tags: [Elasticsearch]
categories: [Search]
---

#### query Section

query section（查询部分）包含Lucene在特定分片上执行的查询树的详细时间。 此查询树的整体结构将类似于你的原始Elasticsearch查询，但可能稍微（或有时非常）不同。 它也将使用相似但不总是相同的命名。 使用我们以前的term查询示例，我们来分析查询部分：

```
"query": [
    {
       "query_type": "BooleanQuery",
       "lucene": "message:search message:test",
       "time": "15.52889800ms",
       "breakdown": {...},            (1)   
       "children": [
          {
             "query_type": "TermQuery",
             "lucene": "message:search",
             "time": "4.938855000ms",
             "breakdown": {...}
          },
          {
             "query_type": "TermQuery",
             "lucene": "message:test",
             "time": "0.5016660000ms",
             "breakdown": {...}
          }
       ]
    }
]
```

- (1) 为了简单起见，省略了breakdown时间内容

基于profile剖析结构，我们可以看到，我们的匹配查询被Lucene重写为具有两个子句（两个持有一个TermQuery）的BooleanQuery。 "query_type"字段显示Lucene类名称，并且通常与Elasticsearch中的等效名称对齐。 "lucene"字段显示查询的Lucene解释文本，可用于帮助区分查询的各个部分（例如"message:search"和"message:test"都是TermQuery），否则将显示相同。

"time"字段显示该查询花费了大约15ms的整个BooleanQuery来执行。记录的时间包括所有的子查询。

"breakdown"字段将详细介绍如何花费时间，我们将在稍后再看一下。最后，"children"数组列出了可能存在的任何子查询。因为我们搜索了两个值（"search test"），我们的BooleanQuery包含两个子项TermQueries。它们具有相同的信息（query_type，time，breakdown等）。Children被允许有自己的children。

##### Timing Breakdown（定时分解）

breakdown组件列出了关于底层Lucene执行的详细时间统计信息：

```
"breakdown": {
  "score": 0,
  "next_doc": 24495,
  "match": 0,
  "create_weight": 8488388,
  "build_scorer": 7016015,
  "advance": 0

}
```

时间列表的时间单位为纳秒，并没有被正常化。 关于总体time的所有注意事项都适用于此。 Breakdown的意图是让你感觉到A）Lucene实际上哪个组件耗时，B）各种组件之间的差异程度。 像整体时间一样，breakdown包括了所有的children的时间。

统计的含义如下：

##### All parameters

值|描述
---|---
create_weight|Lucene中的查询必须能够跨多个IndexSearchers进行重用（将其视为针对特定Lucene索引执行搜索的引擎）。 这使得Lucene陷入困境，因为许多查询需要积累与正在使用的索引相关联的临时状态/统计信息，但是Query授权规定必须是不可变的。 为了解决这个问题，Lucene询问每个查询生成一个Weight对象，该对象充当临时上下文对象，以保持与该特定（IndexSearcher，Query）元组相关联的状态。 weight权重指标显示此过程需要多长时间
build_scorer|此参数显示为查询构建Scorer记分器所需的时间。 Scorer记分器的机制是迭代匹配文档，每个文档生成一个分数（例如"foo"与文档的匹配度）。 注意，这记录了生成Scorer对象所需的时间，实际上并不对文档进行评分。 某些查询具有更快或更慢的Scorer记分器的初始化，具体取决于优化，复杂性等。这还可能显示与缓存相关联的时间（如果启用and/or适用于查询）
next_doc|Lucene方法next_doc返回与查询匹配的下一个文档的文档ID。 该统计显示了确定哪个文档是下一个匹配所需的时间，这个过程根据查询的性质而有很大的不同。 Next_doc是一个专门的advance()形式，它对Lucene中的许多查询更为方便。 它相当于advance(docId()+ 1)
advance|advance是next_doc的底层版本：它的目的也是一样找到下一个匹配文档，但需要调用查询来执行额外的任务，例如identifying和moving past skips等。但是并不是所有查询都可以使用next_doc 因此，advance也记录了这些查询的时间。 连接（例如一个布尔值中的must子句）是典型的advance消费者
matches|一些查询，例如短语查询，使用"Two Phase"过程匹配文档。 首先，文档是"大致"匹配的，如果它大致匹配，则会以更严格（并且昂贵的）进程第二次检查。 第二阶段验证是匹配统计量度量。 例如，短语查询首先通过确保短语中的所有词条都存在于文档中来大致检查文档。 如果所有的词条都存在，那么它将执行第二阶段验证，以确保词条按顺序形成短语，这比仅仅检查词条的存在是相对昂贵的。 因为这个两阶段过程只能由少量查询使用，所以度量统计量通常为零
score|这记录了通过它的Scorer记分器特定文档所花费的时间

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/2.4/_literal_query_literal_section.html

