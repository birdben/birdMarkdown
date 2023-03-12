---
title: "ElasticSearch官网翻译_Scroll"
date: 2017-05-26 00:36:04
tags: [Elasticsearch]
categories: [Search]
---

#### Scroll

<b>
当搜索请求返回结果的单个“页面”时，scroll API可用于从单个搜索请求中取回大量结果（甚至所有结果），其与在传统数据库使用游标方式相同。

scroll不适用于实时用户请求，而适用于处理大量数据，例如， 以便将一个索引的内容重新索引到具有不同配置的新索引中。
</b>

##### 客户端支持scrolling和reindexing

一些官方支持的客户端提供帮助来协助从一个索引到另一个索引的scroll搜索和reindexing：

- Perl : See Search::Elasticsearch::Bulk and Search::Elasticsearch::Scroll
- Python : See elasticsearch.helpers.*

###### 注意

<b>
从scroll请求返回的结果反映了在进行初始搜索请求时索引的状态，如时间上的快照。 对文档（索引，更新或删除）的后续更改只会影响稍后的搜索请求。

为了使用scrolling，初始搜索请求应该在query string（查询字符串）中指定scroll参数，告诉Elasticsearch应该保持“搜索上下文”活动有多长时间（请参阅保持搜索上下文），例如：?scroll=1m。
</b>

```
curl -XGET 'localhost:9200/twitter/tweet/_search?scroll=1m' -d '
{
    "query": {
        "match" : {
            "title" : "elasticsearch"
        }
    }
}
'
```

上述请求的结果包括_scroll_id，它应该传递给scroll API，以便取回下一批结果。

```
curl -XGET (1) 'localhost:9200/_search/scroll' (2) -d'
{
    "scroll" : "1m", (3)
    "scroll_id" : "c2Nhbjs2OzM0NDg1ODpzRlBLc0FXNlNyNm5JWUc1" (4)
}
'
```

<b>

- (1) 可以使用GET或POST。
- (2) URL不应包含index或type名称 - 这些在原始搜索请求中指定。
- (3) scroll参数告诉Elasticsearch将搜索上下文保持为另外1m。
- (4) scroll_id参数

每次调用scroll API会返回下一批结果，直到没有更多结果返回，即命中数组为空。
</b>

为了向后兼容，scroll_id和scroll可以在query string（查询字符串）中传递。 而scroll_id可以在请求正文中传递

```
curl -XGET 'localhost:9200/_search/scroll?scroll=1m' -d 'c2Nhbjs2OzM0NDg1ODpzRlBLc0FXNlNyNm5JWUc1'
```

##### 重要

<b>初始搜索请求和每个后续滚动请求返回一个新的_scroll_id - 只应使用最近的_scroll_id。</b>

###### 注意

如果请求指定聚合，则只有初始搜索响应将包含聚合结果。

###### 注意

<b>Scroll请求具有优化，使排序顺序为_doc时更快。 如果你希望迭代所有文档，无论顺序如何，这是最有效的选项：</b>

```
curl -XGET 'localhost:9200/_search?scroll=1m' -d '
{
  "sort": [
    "_doc"
  ]
}
'
```

##### Keeping the search context alive

<b>
Scroll参数（传递给search请求和每个scroll请求）告诉Elasticsearch应该保持搜索上下文的时间长短。 它的值（例如：1分钟，参阅“Time units”一节）不需要足够长的时间来处理所有数据 - 只需要足够长的时间来处理上一批结果。 每个scroll请求（与scroll参数一起）设置一个新的到期时间。

通常，后台merge合并过程通过将较小的段合并在一起来创建新的更大的段来优化索引，此时较小的段被删除。 此过程在scrolling期间继续，但打开的搜索上下文可防止旧段在其仍在使用中被删除。 这是Elasticsearch如何能够返回初始搜索请求的结果，而不管文档的后续更改如何。
</b>

###### 提示

<b>保持较旧的分段活着意味着需要更多的文件句柄。 确保你已经将节点配置为拥有足够的可用文件句柄。</b> 请参阅“File Descriptors”一节。

你可以使用nodes stats API来检查打开的search contexts（搜索上下文）数量：

```
curl -XGET localhost:9200/_nodes/stats/indices/search?pretty
```

##### Clear scroll API

当超出scroll超时时间，搜索上下文将自动删除。 然而，保持scroll打开有成本，如上一节所述，所以一scroll不再使用clear-scroll API，所以scroll应该被明确地清除：

```
curl -XDELETE localhost:9200/_search/scroll -d '
{
    "scroll_id" : ["c2Nhbjs2OzM0NDg1ODpzRlBLc0FXNlNyNm5JWUc1"]
}'
```

多个scroll IDs可以作为数组传递：

```
curl -XDELETE localhost:9200/_search/scroll -d '
{
    "scroll_id" : ["c2Nhbjs2OzM0NDg1ODpzRlBLc0FXNlNyNm5JWUc1", "aGVuRmV0Y2g7NTsxOnkxaDZ"]
}'
```

所有搜索上下文可以使用_all参数清除：

```
curl -XDELETE localhost:9200/_search/scroll/_all
```

scroll_id也可以作为query string（查询字符串）参数或请求体中传递。 多个scroll IDs可以以逗号分隔的值传递：

```
curl -XDELETE localhost:9200/_search/scroll \
     -d 'c2Nhbjs2OzM0NDg1ODpzRlBLc0FXNlNyNm5JWUc1,aGVuRmV0Y2g7NTsxOnkxaDZ'
```

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/2.4/search-request-scroll.html

