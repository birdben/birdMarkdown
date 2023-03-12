---
title: "ElasticSearch官网翻译_GetSettings"
date: 2017-06-05 14:31:00
tags: [Elasticsearch]
categories: [Search]
---

## Get Settings

Get settings API允许取回索引/索引的设置信息：

```
GET /twitter/_settings
```

### Multiple Indices and Types（多索引和多类型）

Get settings API可用于通过单个调用获取多个索引的设置。 API的一般用法遵循以下语法：host:port/{index}/_settings，其中{index}可以代表索引名称和别名的逗号分隔列表。 要获取所有索引的设置，你可以使用_all替换{index}。 通配符表达式也被支持。 以下是一些例子：

```
GET /twitter,kimchy/_settings

GET /_all/_settings

GET /log_2013_*/_settings
```

### Filtering settings by name（对名称过滤设置）

返回的设置可以使用通配符匹配进行过滤，如下所示：

```
curl -XGET 'http://localhost:9200/2013-*/_settings/index.number_*'
```

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/5.4/indices-get-settings.html
