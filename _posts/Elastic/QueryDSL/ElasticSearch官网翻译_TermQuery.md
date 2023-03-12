---
title: "ElasticSearch官网翻译_TermQuery"
date: 2017-06-22 14:45:08
tags: [Elasticsearch]
categories: [Search]
---

## Term Query

Term Query在倒排索引中查找包含指定的确切词条的文档。 例如：

```
POST _search
{
  "query": {
    "term" : { "user" : "Kimchy" } (1)
  }
}
```

- (1) 在user字段的倒排索引中查找包含确切词条Kimchy的文档。

可以指定boost参数，以使此Term Query具有比另一个查询更高的相关性分数，例如：

```
GET _search
{
  "query": {
    "bool": {
      "should": [
        {
          "term": {
            "status": {
              "value": "urgent",
              "boost": 2.0 (1)
            }
          }
        },
        {
          "term": {
            "status": "normal" (2)
          }
        }
      ]
    }
  }
}
```

- (1) urgent查询子句的boost为2.0，这意味着它的重要程度是normal查询子句的两倍。
- (2) normal查询子句的默认boost为1.0。

######  Why doesn’t the term query match my document?（为什么term query匹配我的文档？）

> 字符串字段可以是text（视为full text全文，如电子邮件的正文）或keyword（被视为确切的值，如电子邮件地址或邮政编码）类型。精确值（如numbers，dates和keywords）在添加到倒排索引的字段中指定的确切值，以使其可搜索。
> 
> 但是，analyzed的text字段。这意味着它们的值首先通过分析器传递以产生词条列表，然后将其添加到倒排索引中。
> 
> 有很多方法可以分析文本：默认的standard analyzer（标准分析器）可以删除大多数标点符号，将文本分解成单个单词，并将其小写化。例如，标准分析器将字符串"Quick Brown Fox!"转换成[quick，brown，fox]。
> 
> 该分析过程使得可以在大块全文中搜索单个单词。
> 
> Term Query在字段的倒排索引中查找确切的词条 - 它不了解该字段的分析器的任何内容。这样可以在keyword字段，numeric或date字段中查找值很有用。查询全文字段时，请改用Match Query，此查询了解字段的分析方式。
> 
> 示范，请尝试下面的示例。首先，创建索引，指定字段映射和索引文档：
> 
> ```
PUT my_index
{
  "mappings": {
    "my_type": {
      "properties": {
        "full_text": {
          "type":  "text" (1)
        },
        "exact_value": {
          "type":  "keyword" (2)
        }
      }
    }
  }
}
> 
PUT my_index/my_type/1
{
  "full_text":   "Quick Foxes!", (3)
  "exact_value": "Quick Foxes!"  (4)
}
```
> (1) full_text字段是text类型，将被分析。
> 
> (2) exact_value字段是keyword类型，不会被分析。
> 
> (3) full_text倒排索引将包含以下词条：[quick，foxes]。
> 
> (4) exact_value倒排索引将包含确切的词条：[Quick Foxes!]。
> 
> 现在，比较Term Query和Match Query的结果：
> 
> ```
GET my_index/my_type/_search
{
  "query": {
    "term": {
      "exact_value": "Quick Foxes!" (1)
    }
  }
}
> 
GET my_index/my_type/_search
{
  "query": {
    "term": {
      "full_text": "Quick Foxes!" (2)
    }
  }
}
> 
GET my_index/my_type/_search
{
  "query": {
    "term": {
      "full_text": "foxes" (3)
    }
  }
}
> 
GET my_index/my_type/_search
{
  "query": {
    "match": {
      "full_text": "Quick Foxes!" (4)
    }
  }
}
```
> (1) 此查询匹配，因为exact_value字段包含确切的词条"Quick Foxes!"
> 
> (2) 此查询不匹配，因为full_text字段仅包含词条"quick"和"foxes"。 它不包含确切的词条"Quick Foxes!"
> 
> (3) 词条"foxes"的Term query与full_text字段匹配。
> 
> (4) full_text字段上的Match Query首先分析query string（查询字符串），然后查找包含"quick"或"foxes"或两者都有的文档。

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/5.4/query-dsl-term-query.html
