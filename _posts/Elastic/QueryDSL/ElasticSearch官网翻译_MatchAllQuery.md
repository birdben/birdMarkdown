---
title: "ElasticSearch官网翻译_MatchAllQuery"
date: 2017-06-13 11:52:37
tags: [Elasticsearch]
categories: [Search]
---

## Match All Query

最简单的查询，它匹配所有文档，给它们全部为_score为1.0。

```
GET /_search
{
    "query": {
        "match_all": {}
    }
}
```

可以使用boost参数更改_score：

```
GET /_search
{
    "query": {
        "match_all": { "boost" : 1.2 }
    }
}
```

### Match None Query

这是与match_all查询相反的查询，它不匹配任何文档。

```
GET /_search
{
    "query": {
        "match_none": {}
    }
}
```

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/5.4/query-dsl-match-all-query.html
