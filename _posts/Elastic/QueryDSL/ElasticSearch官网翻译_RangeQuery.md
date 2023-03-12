---
title: "ElasticSearch官网翻译_RangeQuery"
date: 2017-06-24 16:28:33
tags: [Elasticsearch]
categories: [Search]
---

## Range Query

匹配文档的字段的词条在指定范围内。 Lucene查询的类型取决于字段类型，对于string字段，使用TermRangeQuery，而对于number/date字段，查询是NumericRangeQuery。 以下示例返回所有age在10到20之间的文档：

```
GET _search
{
    "query": {
        "range" : {
            "age" : {
                "gte" : 10,
                "lte" : 20,
                "boost" : 2.0
            }
        }
    }
}
```

range查询接受以下参数：

值|描述
---|---
gte|大于等于
gt|大于
lte|小于等于
lt|小于
boost|设置查询的boost值，默认为1.0

### Ranges on date fields（日期字段范围）

当在date类型的字段上运行range查询时，可以使用"Date Math"一节指定范围：

```
GET _search
{
    "query": {
        "range" : {
            "date" : {
                "gte" : "now-1d/d",
                "lt" :  "now/d"
            }
        }
    }
}
```

#### Date math and rounding（日期计算和四舍五入）

当使用date math将日期四舍五入到最近的day, month, hour等等，舍入后的日期取决于范围的末尾是包含还是排除。

向上舍入移动到舍入范围的最后一毫秒，向下舍入移动到舍入范围的第一毫秒。 例如：

值|描述
---|---
gt|大于日期向上舍入:2014-11-18||/M成为2014-11-30T23:59:59.999，即不包括整个月。
gte|大于或等于日期向下舍入:2014-11-18||/M成为2014-11-01，即包括整个月。
lt|少于日期向下舍入:2014-11-18||/M成为2014-11-01，即不包括整个月。
lte|小于等于日期向上舍入:2014-11-18||/M成为2014-11-30T23:59:59.999，即包括整个月。

#### Date format in range queries（范围查询中的日期格式）

格式化日期将使用默认情况下在date字段上指定的format解析，但可以通过将format参数传递给range查询来覆盖格式化日期：

```
GET _search
{
    "query": {
        "range" : {
            "born" : {
                "gte": "01/01/2012",
                "lte": "2013",
                "format": "dd/MM/yyyy||yyyy"
            }
        }
    }
}
```

#### Time zone in range queries（范围查询中的时区）

可以通过在日期值本身中指定时区（如果format接受）将日期从另一个时区转换为UTC，或者可以使用time_zone参数指定：

```
GET _search
{
    "query": {
        "range" : {
            "timestamp" : {
                "gte": "2015-01-01 00:00:00", (1)
                "lte": "now", (2)
                "time_zone": "+01:00"
            }
        }
    }
}
```

- (1) 此日期将转换为2014-12-31T23:00:00 UTC。
- (2) now不受time_zone参数的影响（日期必须存储为UTC）。

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/5.4/query-dsl-range-query.html
