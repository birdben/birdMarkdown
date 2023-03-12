---
title: "ElasticSearch官网翻译_PostFilter"
date: 2017-05-24 00:43:21
tags: [Elasticsearch]
categories: [Search]
---

#### Post filter

在已经计算了聚合之后，post_filter应用于搜索请求最后的搜索匹配。 其目的最好的例子如下：

想象一下，你正在销售衬衫，用户已经指定了两个过滤器：color:red和brand:gucci。 你只想在搜索结果中显示Gucci制造的红色衬衫。 通常你会用一个bool query：

```
curl -XGET localhost:9200/shirts/_search -d '
{
  "query": {
    "bool": {
      "filter": [
        { "term": { "color": "red"   }},
        { "term": { "brand": "gucci" }}
      ]
    }
  }
}
'
```

但是，你也可以使用faceted navigation（分面导航）来显示用户可以点击的其他选项的列表。 也许你有一个model字段，允许用户将他们的搜索结果限制在红色的Gucci t-shirts或dress-shirts。

这可以用terms aggregation来完成：

```
curl -XGET localhost:9200/shirts/_search -d '
{
  "query": {
    "bool": {
      "filter": [
        { "term": { "color": "red"   }},
        { "term": { "brand": "gucci" }}
      ]
    }
  },
  "aggs": {
    "models": {
      "terms": { "field": "model" } (1)
    }
  }
}
'
```

- (1) 返回Gucci最受欢迎的红色衬衫款式。

<b>
但也许你也想告诉用户其他颜色有多少Gucci衬衫。 如果你只是在color字段中添加terms aggregation，则只会返回颜色为red，因为你的查询只返回Gucci的红色衬衫。

相反，你要在聚合期间包括所有颜色的衬衫，然后将colors过滤器应用于搜索结果。 这是post_filter的目的：
</b>

```
curl -XGET localhost:9200/shirts/_search -d '
{
  "query": {
    "bool": {
      "filter": {
        "term": { "brand": "gucci" } (1)
      }
    }
  },
  "aggs": {
    "colors": {
      "terms": { "field": "color" } (2)
    },
    "color_red": {
      "filter": {
        "term": { "color": "red" } (3)
      },
      "aggs": {
        "models": {
          "terms": { "field": "model" } (4)
        }
      }
    }
  },
  "post_filter": { (5)
    "term": { "color": "red" }
  }
}
'
```

<b>

- (1) 主要查询现在找到Gucci的所有衬衫，不管颜色如何。
- (2) colors agg聚合结果返回Gucci衬衫流行的颜色。
- (3)(4) color_red agg聚合结果将models子聚合限制为红色Gucci衬衫。
- (5) 最后，post_filter从搜索匹配中除去红色以外的颜色。

</b>

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/2.4/search-request-post-filter.html

