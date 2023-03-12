---
title: "ElasticSearch官网翻译_FunctionScoreQuery"
date: 2017-06-29 20:55:49
tags: [Elasticsearch]
categories: [Search]
---

## Function Score Query

function_score允许你修改查询检索到的文档的分数。 这个很有用，举个例子，score function（计分函数）在计算上是昂贵的，并且在已经过滤的一组文档上计算分数就足够了。

要使用function_score，用户必须定义一个查询和一个或多个函数，为查询返回的每个文档计算一个新的分数。

function_score只能使用一个这样的函数：

```
GET /_search
{
    "query": {
        "function_score": {
            "query": { "match_all": {} },
            "boost": "5",
            "random_score": {}, (1)
            "boost_mode":"multiply"
        }
    }
}
```

- (1) 有关支持的功能列表，请参阅Function Score Query。

此外，可以组合几个函数。 在这种情况下，只有当文档与给定的过滤查询匹配时，才可以选择应用该函数

```
GET /_search
{
    "query": {
        "function_score": {
          "query": { "match_all": {} },
          "boost": "5", (1)
          "functions": [
              {
                  "filter": { "match": { "test": "bar" } },
                  "random_score": {}, (2)
                  "weight": 23
              },
              {
                  "filter": { "match": { "test": "cat" } },
                  "weight": 42
              }
          ],
          "max_boost": 42,
          "score_mode": "max",
          "boost_mode": "multiply",
          "min_score" : 42
        }
    }
}
```

- (1) boost提升整个查询。
- (2) 有关支持的功能列表，请参阅Function Score Query。

###### 注意

> 每个函数的过滤查询产生的分数无关紧要。

如果函数中没有使用过滤器，则相当于指定"match_all":{}

首先，每个文档都通过定义的函数进行评分。 参数score_mode指定计算得分如何组合：

值|描述
---|---
multiply|分数相乘（默认）
sum|得分相加
avg|平均值
first|应用具有匹配过滤器的第一个函数
max|使用最大分数
min|使用最小分数

因为分数可以在不同的比例上（例如，decay functions（衰减函数）在0到1之间，除了指定field_value_factor），并且还因为有时函数对分数有不同的影响，所以每个函数的得分可以用用户定义weight（权重）。可以在functions函数数组（上面的示例）中为每个函数定义weight（权重），并与相应函数计算的分数相乘。如果没有任何函数声明给出weight（权重），weight（权重）作为简单地返回weight（权重）的函数。

如果将score_mode设置为avg，则单个分数将被weighted average（加权平均值）组合。例如，如果两个函数返回1和2，并且它们各自的权重为3和4，则它们的分数将被组合为(1*3+2*4)/(3+4)而不是(1*3+2*4)/2。

通过设置max_boost参数，可以将新分数限制为不超过某个限制。 max_boost的默认值为FLT_MAX。

新计算的分数与查询的分数相结合。参数boost_mode定义如何：

值|描述
---|---
multiply|查询得分和函数得分相乘（默认）
replace|只使用函数得分，查询得分被忽略
sum|查询得分和函数得分相加
avg|平均
max|查询得分和函数得分的最大值
min|查询得分和函数得分的最小值

默认情况下，修改分数不会改变哪些文档匹配。 要排除不符合某个分数阈值的文档，可以将min_score参数设置为所需的分数阈值。

###### 注意

> 要使min_score工作，查询返回的所有文档都需要记分，然后逐个过滤掉。

function_score查询提供了几种类型的分数函数。

- script_score
- weight
- random_score
- field_value_factor
- decay functions: gauss, linear, exp

#### Script score（脚本计算得分）

script_score函数允许你包装另一个查询，并可以使用脚本表达式从文档中的其他数字字段值派生的计算来自定义它的评分。 这是一个简单的例子：

```
GET /_search
{
    "query": {
        "function_score": {
            "query": {
                "match": { "message": "elasticsearch" }
            },
            "script_score" : {
                "script" : {
                  "inline": "Math.log(2 + doc['likes'].value)"
                }
            }
        }
    }
}
```

在不同的脚本字段值和表达式之上，_score脚本参数可用于基于包装查询来检索分数。

脚本编译被缓存以便更快的执行。 如果脚本有需要参数的考虑，则最好重用相同的脚本，并提供参数：

```
GET /_search
{
    "query": {
        "function_score": {
            "query": {
                "match": { "message": "elasticsearch" }
            },
            "script_score" : {
                "script" : {
                    "params": {
                        "a": 5,
                        "b": 1.2
                    },
                    "inline": "params.a / Math.pow(params.b, doc['likes'].value)"
                }
            }
        }
    }
}
```

请注意，与custom_score查询不同，查询的分数与脚本评分的结果相乘。 如果你想禁止这一点，设置"boost_mode":"replace"

#### Weight（权重影响得分）

weight（权重）允许你将分数乘以提供的weight（权重）来计算评分。 有时可能需要这样做，因为在特定查询上设置的boost值被标准化，而对于score function是无效的。 number值为float类型。

```
"weight" : number
```

#### Random（随机得分）

random_score使用_uid字段的散列生成分数，并使用seed进行变化。 如果未指定seed，则使用当前时间。

###### 注意

> 使用此功能将加载_uid的字段数据，这可能是内存密集型操作，因为值是唯一的。

```
GET /_search
{
    "query": {
        "function_score": {
            "random_score": {
                "seed": 10
            }
        }
    }
}
```

#### Field Value factor（字段值因子）

field_value_factor函数允许你使用文档中的字段来影响分数。 它类似于使用script_score函数，但是它避免了脚本的开销。 如果在多值字段上使用，则只能在计算中使用字段的第一个值。

例如，假设你有一个用数字类型popularity字段的文档被索引，并希望使用该字段影响文档的分数，这样做的示例将如下所示：

```
GET /_search
{
    "query": {
        "function_score": {
            "field_value_factor": {
                "field": "likes",
                "factor": 1.2,
                "modifier": "sqrt",
                "missing": 1
            }
        }
    }
}
```

这将转化为以下公式得分：

sqrt(1.2 * doc['likes'].value)

field_value_factor函数有很多选项：

- field : 要从文档中提取的字段。
- factor : 将字段值乘以的可选因子，默认为1。
- modifier : 应用于字段值的修饰符可以是：none，log，log1p，log2p，ln，ln1p，ln2p，square，sqrt或reciprocal中的一个。 默认为none。

修饰符|描述
---|---
none|不要对字段值应用任何乘数
log|取字段值的对数
log1p|将1添加到字段值并取对数
log2p|将2添加到字段值并取对数
ln|取字段值的自然对数
ln1p|添加1到字段值并采取自然对数
ln2p|添加2到字段值并采取自然对数
square|字段值的平方（乘以它本身）
sqrt|取字段值的平方根
reciprocal|字段值的倒数，与1/x相同，其中x是字段的值

- missing : 如果文档没有该字段，则使用的值。 modifier和factor仍然适用于它，就像从文档中读取一样。

```
记住采用log()为0，或平方根为负数是一个非法操作，一个异常将被抛出。 一定要使用范围过滤器来限制字段值避免这种情况，或使用`log1p`和`ln1p`。
```

#### Decay functions（衰减函数）

Decay functions（衰减函数）使用函数对文档进行计算评分，该函数根据用户给定原始文档的数字字段值的距离而衰减。 这与range query类似，但具有平滑的边缘而不是方框。

要在具有数字字段的查询上使用距离评分，用户必须为每个字段定义origin（原点）和scale（比例）。 需要origin（原点）来定义计算距离的"central point"（中心点），以及定义衰减速率的scale（比例）。 衰减函数指定为

```
"DECAY_FUNCTION": { (1)
    "FIELD_NAME": { (2)
          "origin": "11, 12",
          "scale": "2km",
          "offset": "0km",
          "decay": 0.33
    }
}
```

- (1) DECAY_FUNCTION应该是linear，exp或者gauss之一。
- (2) 指定的字段必须是numeric，date或geo-point字段。

在上面的例子中，该字段是geo_point，并且可以以geo格式提供origin。 在这种情况下，必须使用单位给定scale（比例）和offset（偏移量）。 如果你的字段是日期字段，则可以将scale（比例）和offset（偏移量）设置为天数，周数等。 例：

```
GET /_search
{
    "query": {
        "function_score": {
            "gauss": {
                "date": {
                      "origin": "2013-09-17", (1)
                      "scale": "10d",
                      "offset": "5d", (2)
                      "decay" : 0.5 (3)
                }
            }
        }
    }
}
```

- (1) origin的日期格式取决于映射中定义的format。 如果没有定义origin，则使用当前时间。
- (2)(3) offset和decay参数是可选的。

 * origin : 用于计算距离的原点。 numeric类型字段必须提供number数字，date类型字段提供date日期，geo类型字段提供geo point地理位置。 geo和numeric类型字段是必需的。 对于date字段，默认为now。 原始数据支持日期数学（例如现在为1h）。

 * scale : 所有类型都需要定义与origin（原点） + offset（偏移量）的距离，其中计算得分将等于decay（衰减）参数。 对于geo字段：可以定义为数字 + 单位（1km, 12m,…）。 默认单位是米。 对于date字段：可以定义为数字 + 单位（"1h", "10d",…）。 默认单位为毫秒。 numeric字段：任意数字。

 * offset : 如果定义了offset（偏移量），衰减函数将仅计算距离大于定义offset（偏移量）的文档的衰减函数。 默认值为0。

 * decay : decay（衰减）参数定义了如何按scale（比例）给出的距离对文档进行评分。 如果没有定义decay（衰减），则距离scale（比例）的文档将得分为0.5。

在第一个示例中，你的文档可能代表酒店并包含地理位置字段。 你想要计算一个衰减函数，取决于酒店距离给定位置的距离。 你可能不会立即看到gauss function（高斯函数）的选择范围，但你可以这样说：“距离所期望位置2公里的距离，分数应减少到三分之一。” 然后将自动调整参数"scale"，对于距离所期望位置2公里的酒店，得分函数计算为0.33分。

在第二个例子中，2013-09-12和2013-09-22之间的字段值的文档的权重将为1.0，文档距离该日期（2013-09-17）15天的权重为0.5。

#### Supported decay functions

暂不翻译

#### Multi-values fields（多值字段）

如果用于计算衰减的字段包含多个值，则每个默认值选择最接近原点的值来确定距离。 这可以通过设置multi_value_mode来更改。

值|描述
---|---
min|取最小距离
max|取最大距离
avg|取平均距离
sum|取所有距离之和

例子：

```
    "DECAY_FUNCTION": {
        "FIELD_NAME": {
              "origin": ...,
              "scale": ...
        },
        "multi_value_mode": "avg"
    }
```

#### Detailed example

假设你正在某个城镇寻找一家酒店。你的预算有限，此外你希望酒店靠近市中心，所以酒店距离所期望位置越远，你不太可能入住。

你希望根据你的标准（例如，"hotel, Nancy, non-smoker"）的查询结果在距离市中心的距离以及价格方面进行评分。

直观地，你想将镇中心定义为原点，也许你愿意从酒店步行2公里到市中心。在这种情况下，你的位置字段的origin（原点）是市中心，scale（变化程度）是~2公里。

如果你的预算很低，与昂贵的东西相比你可能会喜欢便宜的东西。对于价格字段，原点将为0欧元，而且其scale取决于你愿意支付多少，例如20欧元。

在这个例子中，酒店的价格字段可能称为"price"，这个酒店的坐标字段是"location"。

在这种情况下，price的函数将是

```
"gauss": { (1)
    "price": {
          "origin": "0",
          "scale": "20"
    }
}
```

- (1) 该衰减函数也可以是linear（线性的）或exp（指数的）。

对于location：

```
"gauss": { (1)
    "location": {
          "origin": "11, 12",
          "scale": "2km"
    }
}
```

- (1) 该衰减函数也可以是linear（线性的）或exp（指数的）。

假设你想在原始分数上乘以这两个函数，请求将如下所示：

```
GET /_search
{
    "query": {
        "function_score": {
          "functions": [
            {
              "gauss": {
                "price": {
                  "origin": "0",
                  "scale": "20"
                }
              }
            },
            {
              "gauss": {
                "location": {
                  "origin": "11, 12",
                  "scale": "2km"
                }
              }
            }
          ],
          "query": {
            "match": {
              "properties": "balcony"
            }
          },
          "score_mode": "multiply"
        }
    }
}
```

接下来，我们展示如何计算出的分数对于三种可能的衰减函数中的每一种来说都是如此。

##### Normal decay, keyword gauss

暂不翻译

##### Exponential decay, keyword exp

暂不翻译

##### Linear decay, keyword linear

暂不翻译

#### Supported fields for decay functions（支持衰减功能的字段）

仅支持numeric，date和geo-point字段。

#### What if a field is missing?（如果没有某一个字段怎么办？）

如果文档中缺少数字字段，则该函数将返回1。

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/5.4/query-dsl-function-score-query.html
