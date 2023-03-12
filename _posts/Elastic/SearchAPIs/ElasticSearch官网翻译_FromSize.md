---
title: "ElasticSearch官网翻译_FromSize"
date: 2017-05-24 14:27:05
tags: [Elasticsearch]
categories: [Search]
---

#### From / Size

可以使用from和size参数来对结果分页处理。 from参数定义与要获取的第一个结果的偏移量。 size参数允许你配置要返回的最大命中数。

虽然from和size可以设置为request请求参数，但也可以在search body搜索体内设置。 从默认值为0，并且size的默认值为10。

```
{
    "from" : 0, "size" : 10,
    "query" : {
        "term" : { "user" : "kimchy" }
    }
}
```

请注意，from + size不能超过index.max_result_window索引设置，默认值为10,000。 请参阅Scroll API以获得更有效的方式进行深层滚动。

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/2.4/search-request-from-size.html