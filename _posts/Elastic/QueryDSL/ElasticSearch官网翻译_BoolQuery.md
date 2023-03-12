---
title: "ElasticSearch官网翻译_BoolQuery"
date: 2017-06-29 18:09:36
tags: [Elasticsearch]
categories: [Search]
---

## Bool Query

查询匹配的文档与其他查询的布尔组合匹配。 bool查询映射到Lucene BooleanQuery。 它使用一个或多个布尔子句构建，每个子句都带有类型的事件。 事件类型有：

值|描述
---|---
must|子句（查询）必须出现在匹配的文档中，并有助于得分。
filter|子句（查询）必须出现在匹配的文档中。 但是与must不同的是查询的分数将被忽略。 过滤器子句在filter context（过滤器上下文）中执行，这意味着计分被忽略，并且子句被考虑用于缓存。
should|子句（查询）应该出现在匹配的文档中。 如果bool查询在query context（查询上下文）中，并且有must或filter子句，则即使没有should查询匹配，文档也将匹配bool查询。 在这种情况下，这些子句仅用于影响分数。 如果bool查询是一个filter context（过滤器上下文）中或没有must和filter子句，则至少有一个should查询必须与文档匹配才能匹配bool查询。 这种行为可能由minimum_should_match参数的设置明确控制。
must_not|子句（查询）不能出现在匹配的文档中。 子句在filter context（过滤器上下文）中被执行，这意味着评分被忽略，并且子句被考虑用于缓存。 因为评分被忽略，所有返回的文档的分数为0。

###### 重要：Bool query in filter context（Bool在过滤器上下文查询）

> 如果此查询在过滤器上下文中使用，并且它具有should的子句，则至少需要一个should子句匹配。

bool查询还支持disable_coord参数（默认为false），它改变了classic相似度如何计算bool查询的分数。 基本上，坐标相似性基于文档包含的所有查询词条的分数来计算分数因子。 有关更多详细信息，请参阅Lucene BooleanQuery。

bool查询采用more-matches-is-better的方法，因此来自每个匹配的must或should子句的分数将被加在一起，为每个文档提供最终的_score。

```
POST _search
{
  "query": {
    "bool" : {
      "must" : {
        "term" : { "user" : "kimchy" }
      },
      "filter": {
        "term" : { "tag" : "tech" }
      },
      "must_not" : {
        "range" : {
          "age" : { "gte" : 10, "lte" : 20 }
        }
      },
      "should" : [
        { "term" : { "tag" : "wow" } },
        { "term" : { "tag" : "elasticsearch" } }
      ],
      "minimum_should_match" : 1,
      "boost" : 1.0
    }
  }
}
```

#### Scoring with bool.filter（用bool.filter打分）

filter元素下指定的查询对计分没有影响 - 分数返回为0。分数仅受指定的query查询的影响。 例如，以下所有三个查询返回所有文档，其中status字段包含词条active。

此第一个查询为所有文档分配得分为0，因为没有指定评分查询：

```
GET _search
{
  "query": {
    "bool": {
      "filter": {
        "term": {
          "status": "active"
        }
      }
    }
  }
}
```

这个bool查询有一个match_all查询，它将所有文档的分数赋予1.0。

```
GET _search
{
  "query": {
    "bool": {
      "must": {
        "match_all": {}
      },
      "filter": {
        "term": {
          "status": "active"
        }
      }
    }
  }
}
```

此constant_score查询的行为与上述第二个示例完全相同。 constant_score查询为所有与过滤器匹配的文档分配1.0分。

```
GET _search
{
  "query": {
    "constant_score": {
      "filter": {
        "term": {
          "status": "active"
        }
      }
    }
  }
}
```

#### Using named queries to see which clauses matched（使用命名查询查看哪些子句匹配）

如果你需要知道bool查询中哪些子句与查询返回的文档匹配，则可以使用named queries（命名查询）为每个子句分配一个名称。

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/5.4/query-dsl-bool-query.html
