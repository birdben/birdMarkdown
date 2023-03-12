---
title: "ElasticSearch官网翻译_RolloverIndex"
date: 2017-06-03 16:20:51
tags: [Elasticsearch]
categories: [Search]
---

## Rollover Index

当现有索引被认为太大或太旧时，rollover index API会将别名滚动到新的索引。

这个API接受单个alias别名和conditions列表。 别名只能指向一个索引。 如果索引满足指定的条件，则创建新的索引，并将别名切换到指向新的索引。

```
PUT /logs-000001 (1)
{
  "aliases": {
    "logs_write": {}
  }
}

# Add > 1000 documents to logs-000001

POST /logs_write/_rollover (2)
{
  "conditions": {
    "max_age":   "7d",
    "max_docs":  1000
  }
}
```

- (1) 使用别名logs_write创建一个名为logs-0000001的索引。
- (2) 如果logs_write指向的索引是在7天或更多天前创建的，或者包含1,000个或更多文档，则会创建logs-000002索引，并更新logs_write别名以指向logs-000002。

上述请求可能会返回以下响应：

```
{
  "acknowledged": true,
  "shards_acknowledged": true,
  "old_index": "logs-000001",
  "new_index": "logs-000002",
  "rolled_over": true, (1)
  "dry_run": false, (2)
  "conditions": { (3)
    "[max_age: 7d]": false,
    "[max_docs: 1000]": true
  }
}
```

- (1) 该索引是否滚动。
- (2) 是否干运行滚动。
- (3) 每个滚动条件的结果。

### Naming the new index（命名新索引）

如果现有索引的名称以"-"和数字结尾。 例如：logs-000001 - 则新索引的名称将遵循相同的模式，增加数字（logs-000002）。 无论旧索引名称如何，编号为零填充长度为6。

如果旧名称与此模式不匹配，则必须如下所示指定新索引的名称：

```
POST /my_alias/_rollover/my_new_index_name
{
  "conditions": {
    "max_age":   "7d",
    "max_docs":  1000
  }
}
```

### Using date math with the rollover API（使用日期计算滚动API）

使用日期计算来根据索引滚动的日期来命名滚动索引是有用的，例如logstash-2016.02.03。 rollover API支持日期计算，但要求索引名称以一个破折号后跟一个数字，例如 logstash-2016.02.03-1，每次索引被滚动时都会增加。 例如：

```
# PUT /<logs-{now/d}-1> with URI encoding:
PUT /%3Clogs-%7Bnow%2Fd%7D-1%3E (1)
{
  "aliases": {
    "logs_write": {}
  }
}

PUT logs_write/log/1
{
  "message": "a dummy log"
}

POST logs_write/_refresh

# Wait for a day to pass

POST /logs_write/_rollover (2)
{
  "conditions": {
    "max_docs":   "1"
  }
}
```

- (1) 创建一个以今天的日期命名的索引（例如）logs-2016.10.31-1
- (2) 以今天的日期滚动到新的索引，例如 logs-2016.10.31-000002如果立即运行，或logs-2016.11.01-000002如果在24小时后运行

然后可以按照date math documentation中的描述来引用这些索引。 例如，要搜索过去三天创建的索引，你可以执行以下操作：

```
# GET /<logs-{now/d}-*>,<logs-{now/d-1d}-*>,<logs-{now/d-2d}-*>/_search
GET /%3Clogs-%7Bnow%2Fd%7D-*%3E%2C%3Clogs-%7Bnow%2Fd-1d%7D-*%3E%2C%3Clogs-%7Bnow%2Fd-2d%7D-*%3E/_search
```

### Defining the new index（定义新索引）

新索引的settings设置，mappings映射和aliases别名取自任何匹配的index templates（索引模板）。 此外，你可以在请求主体中指定settings设置，mappings映射和aliases别名，就像create index API一样。 请求中指定的值将覆盖匹配索引模板中设置的任何值。 例如，以下rollover请求将覆盖index.number_of_shards设置：

```
PUT /logs-000001
{
  "aliases": {
    "logs_write": {}
  }
}

POST /logs_write/_rollover
{
  "conditions" : {
    "max_age": "7d",
    "max_docs": 1000
  },
  "settings": {
    "index.number_of_shards": 2
  }
}
```

### Dry run（干运行）

rollover API支持dry_run模式，可以在不执行实际滚动的情况下检查请求条件：

```
PUT /logs-000001
{
  "aliases": {
    "logs_write": {}
  }
}

POST /logs_write/_rollover?dry_run
{
  "conditions" : {
    "max_age": "7d",
    "max_docs": 1000
  }
}
```

### Wait For Active Shards（等待活动分片）

因为滚动操作会创建一个新的索引以滚动到（新索引上），索引创建上的wait_for_active_shards设置也适用于滚动操作。

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/5.4/indices-rollover-index.html
