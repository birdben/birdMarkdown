---
title: "ElasticSearch官网翻译_precisionStep"
date: 2017-05-20 13:05:27
tags: [Elasticsearch]
categories: [Search]
---

#### precision_step（精度级别）

<b>大多数numeric（数字类型）索引额外的terms（词条）表示每个数字的数字范围，使得range queries（范围查询）更快。</b> 例如，此范围查询：

```
  "range": {
    "number": {
      "gte": 0
      "lte": 321
    }
  }
```

可能在内部执行的terms query（精确查询）如下所示：

```
 "terms": {
    "number": [
      "0-255",
      "256-319"
      "320",
      "321"
    ]
  }
```

<b>这些额外的terms（词条）大大减少了必须检查的terms（词条）数量，代价是增加了磁盘空间。</b>

precision_step的默认值取决于数字字段的类型：

类型|精度
---|---
long, double, date, ip|16（3个额外词条）
integer, float, short|8（3个额外词条）
byte|2147483647（0个额外词条）
token_count|32（0个额外词条）

precision_step设置的值表示应该压缩成额外term（词条）的位数。 long值由64位组成，因此precision_step为16，结果如下：

类型|精度
---|---
Bits 0-15|value & 1111111111111111 0000000000000000 0000000000000000 0000000000000000
Bits 0-31|value & 1111111111111111 1111111111111111 0000000000000000 0000000000000000
Bits 0-47|value & 1111111111111111 1111111111111111 1111111111111111 0000000000000000
Bits 0-63|value


参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/2.4/precision-step.html
