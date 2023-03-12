---
title: "ElasticSearch官网翻译_Suggesters"
date: 2017-05-27 01:53:16
tags: [Elasticsearch]
categories: [Search]
---

#### Suggesters

suggest功能通过使用suggester来提供基于提供的文本的类似的词条。 部分suggest功能仍在开发中。

<b>suggest请求部分通过_search请求在query查询部分旁边定义，或通过REST _suggest endpoint。</b>

```
curl -s -XPOST 'localhost:9200/_search' -d '{
  "query" : {
    ...
  },
  "suggest" : {
    ...
  }
}'
```

<b>
对_suggest endpoint执行的suggest请求应该忽略周围的suggest元素，该元素仅在suggest请求是搜索的一部分时才使用。
</b>

```
curl -XPOST 'localhost:9200/_suggest' -d '{
  "my-suggestion" : {
    "text" : "the amsterdma meetpu",
    "term" : {
      "field" : "body"
    }
  }
}'
```

每个请求可以指定几个建议。 每个建议用任意名称标识。 在下面的例子中，请求两个建议。 my-suggest-1和my-suggest-2的建议都使用词条suggester，但具有不同的text。

```
"suggest" : {
  "my-suggest-1" : {
    "text" : "the amsterdma meetpu",
    "term" : {
      "field" : "body"
    }
  },
  "my-suggest-2" : {
    "text" : "the rottredam meetpu",
    "term" : {
      "field" : "title"
    }
  }
}
```

以下suggest响应示例包括对my-suggest-1和my-suggest-2的建议响应。 每个建议部分都包含条目。 每个条目实际上是来自建议文本的标记，并且包含建议条目文本，原始建议文本中的起始偏移量和长度以及是否找到任意数量的选项。

```
{
  ...
  "suggest": {
    "my-suggest-1": [
      {
        "text" : "amsterdma",
        "offset": 4,
        "length": 9,
        "options": [
           ...
        ]
      },
      ...
    ],
    "my-suggest-2" : [
      ...
    ]
  }
  ...
}
```

每个选项数组包含一个选项对象，其中包含建议文本，其文档频率和得分与建议条目文本相比。 分数的含义取决于使用的suggester。 Term suggester的得分是基于编辑距离。

```
"options": [
  {
    "text": "amsterdam",
    "freq": 77,
    "score": 0.8888889
  },
  ...
]
```

##### Global suggest text（全局建议文本）

为了避免重复建议文本，可以定义全局文本。 在下面的示例中，建议文本在全局范围内定义，适用于my-suggest-1和my-suggest-2的建议。

```
"suggest" : {
  "text" : "the amsterdma meetpu",
  "my-suggest-1" : {
    "term" : {
      "field" : "title"
    }
  },
  "my-suggest-2" : {
    "term" : {
      "field" : "body"
    }
  }
}
```

上述示例中的建议文本也可以指定为建议特定选项。 建议级别上指定的建议文本将覆盖全局级别的建议文本。

##### Other suggest example（其他建议的例子）

在下面的例子中，我们要求对以下建议文字提出建议：devloping distibutd saerch在title字段中，建议文本中每个词条最多可以有3个建议。 请注意，在本示例中，我们将size设置为0。这不是必需的，而是一个不错的优化。 建议在query查询阶段收集，在我们只关心建议（所以没有命中）的情况下，我们不需要执行fetch提取阶段。

```
curl -s -XPOST 'localhost:9200/_search' -d '{
  "size": 0,
  "suggest" : {
    "my-title-suggestions-1" : {
      "text" : "devloping distibutd saerch engies",
      "term" : {
        "size" : 3,
        "field" : "title"
      }
    }
  }
}'
```

上述请求可以产生如下面的代码示例中所述的响应。 正如你可以看到，如果我们采用每个建议条目的第一个建议选项，我们获取到"developing distributed search engines"如下的结果。

```
{
  ...
  "suggest": {
    "my-title-suggestions-1": [
      {
        "text": "devloping",
        "offset": 0,
        "length": 9,
        "options": [
          {
            "text": "developing",
            "freq": 77,
            "score": 0.8888889
          },
          {
            "text": "deloping",
            "freq": 1,
            "score": 0.875
          },
          {
            "text": "deploying",
            "freq": 2,
            "score": 0.7777778
          }
        ]
      },
      {
        "text": "distibutd",
        "offset": 10,
        "length": 9,
        "options": [
          {
            "text": "distributed",
            "freq": 217,
            "score": 0.7777778
          },
          {
            "text": "disributed",
            "freq": 1,
            "score": 0.7777778
          },
          {
            "text": "distribute",
            "freq": 1,
            "score": 0.7777778
          }
        ]
      },
      {
        "text": "saerch",
        "offset": 20,
        "length": 6,
        "options": [
          {
            "text": "search",
            "freq": 1038,
            "score": 0.8333333
          },
          {
            "text": "smerch",
            "freq": 3,
            "score": 0.8333333
          },
          {
            "text": "serch",
            "freq": 2,
            "score": 0.8
          }
        ]
      },
      {
        "text": "engies",
        "offset": 27,
        "length": 6,
        "options": [
          {
            "text": "engines",
            "freq": 568,
            "score": 0.8333333
          },
          {
            "text": "engles",
            "freq": 3,
            "score": 0.8333333
          },
          {
            "text": "eggies",
            "freq": 1,
            "score": 0.8333333
          }
        ]
      }
    ]
  }
  ...
}
```

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/2.4/search-suggesters.html

