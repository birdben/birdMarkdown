---
title: "Elasticsearch学习——倒排索引"
date: 2017-05-14 12:46:34
tags: [Elasticsearch]
categories: [Search]
---

### 倒排索引

Elasticsearch使用一种称为倒排索引的结构，它适用于快速的全文搜索。一个倒排索引由文档中所有不重复词条的列表构成，对于其中每个词条，有一个包含它的文档列表。

为了创建倒排索引，我们首先将每个文档的字段拆分成单独的词条（terms 或 tokens），创建一个包含所有不重复词条的排序列表，然后列出每个词条出现在哪个文档。这里我们暂不详细说明相似性算法。

举例说明倒排索引，新创建一个my_index索引，分别有四个属性name，say，job，say，my_index的mapping如下：

```
{
  "my_index" : {
    "mappings" : {
      "my_type" : {
        "properties" : {
          "name" : {
            "type" : "string",
            "index" : "no",
            "store" : false
          },
          "job" : {
            "type" : "string",
            "index" : "analyzed",
	         "store" : false
          },
          "location" : {
            "type" : "string",
            "index" : "not_analyzed",
            "store" : false
          },
          "say" : {
            "type" : "string",
            "index" : "analyzed",
            "store" : true
          }
        }
      }
    }
  }
}
```

新建两个文档doc1和doc2

```
$ curl -XPUT 'http://localhost:9200/my_index/my_type/1' -d '
{
  "name": "bird",
  "job": "dev Manager",
  "location": "china Shanghai",
  "say": "hello world"
}'

$ curl -XPUT 'http://localhost:9200/my_index/my_type/2' -d '
{
  "name": "birdben",
  "job": "test Manager",
  "location": "china Beijing",
  "say": "hello birdben"
}'
```

先来分析一下mapping中设置的四个字段name，job，location，say：

- name : 
 * index:no，代表name字段不索引（不可被搜索），也就是不会生成倒排索引
 * store:false，代表文档的name字段的内容是不会与_source分开存储并检索的
- job : 
 * index:anaylzed，代表job字段需要被分词索引（先根据job字段的内容分词处理生成词条，根据词条对该词条出现的文档进行索引，可以根据词条进行全文检索），也就是会生成倒排索引
 * store:false，代表文档的job字段的内容是不会与_source分开存储并检索的
- location : 
 * index:not_anaylzed，代表location字段需要不被分词索引（location中的内容不能被分词处理，需要将location的内容整体作为索引，不能根据词条进行全文检索），也就是会生成倒排索引（但是索引的词条就是location的内容整体，而不是分词后的词条）
 * store:false，代表文档的location字段的内容是不会与_source分开存储并检索的
- say : 
 * index:anaylzed，代表say字段需要被分词索引（先根据say字段的内容分词处理生成词条，根据词条对该词条出现的文档进行索引，可以根据词条进行全文检索），也就是会生成倒排索引
 * store:true，代表文档的say字段的内容是需要与_source分开存储并检索的（一般需要对某个字段进行高亮显示才会将store设置为true）。

##### 倒排索引

针对上面的mapping设置，每个字段就会生成如下的倒排索引：

- name字段：无，因为index设置为no

- job字段：

term|doc1|doc2
---|---|---
dev|√| 
manager|√|√
test| |√
---|---|---
Total|2|2

- location字段：

term|doc1|doc2
---|---|---
china Beijing| |√
china Shanghai|√| 
---|---|---
Total|1|1

- say字段：

term|doc1|doc2
---|---|---
birdben| |√
hello|√|√
world|√| 
---|---|---
Total|2|2

#### 查询

##### name字段查询

无论是term还是match查询都无法查出结果，因为name字段没有索引

```
# 没有结果
$ curl -XPOST 'http://localhost:9200/my_index/_search?pretty' -d '{
	"query": {
		"match": {
	      "name": "bird" 
	    }
	}	
}'

# 没有结果
$ curl -XPOST 'http://localhost:9200/my_index/_search?pretty' -d '{
	"query": {
		"term": {
	      "name": "bird" 
	    }
	}	
}'
```

##### job字段查询

这里搜索词是"dev"和"test"使用term和match的结果都是一样的，因为倒排索引中的词条就是"dev"和"test"，但是如果搜索词是"Dev"有大小写区别，term就搜索不出来结果，match就能搜索出来结果。

```
# 有结果，经过分析处理后搜索词仍然是dev，在倒排索引中能够找到
$ curl -XPOST 'http://localhost:9200/my_index/_search?pretty' -d '{
	"query": {
		"match": {
	      "job": "dev" 
	    }
	}	
}'

# 有结果，虽然倒排索引中的词条是dev，而不是Dev，但是match是会对搜索词进行处理的，因为job字段的mapping设置analyzed的，默认搜索job字段match会将搜索词进行分析处理（分词处理，去掉停止词，大小写转换），所以经过分析处理后从Dev变成dev，在倒排索引中能够找到dev
$ curl -XPOST 'http://localhost:9200/my_index/_search?pretty' -d '{
	"query": {
		"match": {
	      "job": "Dev" 
	    }
	}	
}'
```

```
# 有结果，因为倒排索引中词条就是dev
$ curl -XPOST 'http://localhost:9200/my_index/_search?pretty' -d '{
	"query": {
		"term": {
	      "job": "dev" 
	    }
	}	
}'

# 没有结果，因为倒排索引中词条就是dev，而不是Dev，term是不会对搜索词进行分析处理的
$ curl -XPOST 'http://localhost:9200/my_index/_search?pretty' -d '{
	"query": {
		"term": {
	      "job": "Dev" 
	    }
	}	
}'
```

这个通过搜索Manager也能够看出来，虽然doc1和doc2中的job属性分别是"dev Manager"和"test Manager"（注意:Manager是大写的），但实际上生成的倒排索引的词条却是小写的"manager"，因为在索引doc1和doc2时就会进行处理（分词处理，去掉停止词，大小写转换）。

```
# 没有结果，索引时对job属性的内容进行处理（分词处理，去掉停止词，大小写转换）后，Manager已经变成manager，所以term查询没有结果
$ curl -XPOST 'http://localhost:9200/my_index/_search?pretty' -d '{
	"query": {
		"term": {
	      "job": "Manager" 
	    }
	}	
}'

# 有结果，所以term查询"manager"是有结果的
$ curl -XPOST 'http://localhost:9200/my_index/_search?pretty' -d '{
	"query": {
		"term": {
	      "job": "manager" 
	    }
	}	
}'
```

这里直接term精确查询"dev Manager"和"test Manager"也是没有结果的，也是因为在索引doc1和doc2时就会进行处理（分词处理，去掉停止词，大小写转换），原来的内容已经被拆分成词条了。

```
# 没有结果
$ curl -XPOST 'http://localhost:9200/my_index/_search?pretty' -d '{
	"query": {
		"term": {
	      "job": "test Manager" 
	    }
	}	
}'
```

###### 重要

> 你只能搜索在索引中出现的词条，所以索引文本和查询字符串必须标准化为相同的格式。

#### location字段查询

上面使用term查询"test Manager"没有结果是因为job字段的index设置为analyzed，这里location字段的index设置为not_analyzed，代表索引时不会进行处理（分词处理，去掉停止词，大小写转换），所以理解为生成的倒排索引的词条就是location字段的内容

```
# 有结果
$ curl -XPOST 'http://localhost:9200/my_index/_search?pretty' -d '{
	"query": {
		"term": {
	      "location": "china Beijing" 
	    }
	}	
}'

# 有结果，因为location字段的index设置为not_analyzed，所以match查询的时候不会像设置为analyzed一样进行处理（分词处理，去掉停止词，大小写转换），所以就是直接对location的内容的在倒排索引中查询，而倒排索引中的词条也是location的内容，所以就能够查询到结果
$ curl -XPOST 'http://localhost:9200/my_index/_search?pretty' -d '{
	"query": {
		"match": {
	      "location": "china Beijing" 
	    }
	}	
}'
```

#### say字段查询

```
$ curl -XGET 'http://localhost:9200/my_index/my_type/1'
$ curl -XGET 'http://localhost:9200/my_index/my_type/2'

# 可以指定只取回say的字段
$ curl -XPOST 'http://localhost:9200/my_index/_search?pretty' -d '{
	"fields": ["say"]
}'

# 只单独取回say字段的内容
$ curl -XPOST 'http://localhost:9200/my_index/_search?pretty' -d '{
	"query": {
		"match":{
			"say":"birdben"
		}
	},
	"fields": ["say"]
}'

# 当然也可以检索其他字段，但是指定取回的字段只有say没有_source字段
$ curl -XPOST 'http://localhost:9200/my_index/_search?pretty' -d '{
	"query": {
		"match":{
			"job":"test"
		}
	},
	"fields": ["say"]
}'

# 可以指定取回store的字段
$ curl -XPOST 'http://localhost:9200/my_index/_search?pretty' -d '{
	"query": {
		"match":{
			"job":"test"
		}
	},
	"fields": ["_source", "say"]
}'
```

#### Term和Match查询的区别

通过上面的例子可以看出term和match查询的区别

- term : 基于词条的查询，低级查询(Low-level Queries)
 * term是在单一词条上进行操作，只会在倒排索引中寻找该词条的精确匹配。一个针对词条Foo的term查询会在倒排索引中寻找该词条的精确匹配(Exact term)，然后对每一份含有该词条的文档通过TF/IDF进行相关度_score的计算。
- match : 全文查询，高级查询(High-level Queries)
 * 如果你使用它们去查询一个date或者integer字段，它们会将查询字符串分别当做日期或者整型数。
 * 如果你查询一个精确值(not_analyzed)字符串字段，它们会将整个查询字符串当做一个单独的词条。
 * 但是如果你查询了一个全文字段(analyzed)，它们会首先将查询字符串传入到合适的解析器，用来得到需要查询的词条列表。一旦查询得到了一个词条列表，它就会使用列表中的每个词条来执行合适的低级查询（term查询），然后将得到的结果进行合并，最终产生每份文档的相关度分值。

这里总结了一下Elasticsearch的mapping中指定字段的index和store属性的用法，以及指定index对应生成的倒排索引，还分析了term和match查询的区别，修正了自己以前理解正确的地方。

参考文章：

- https://www.elastic.co/guide/cn/elasticsearch/guide/current/inverted-index.html
- https://www.elastic.co/guide/en/elasticsearch/reference/2.4/mapping-store.html
- http://blog.csdn.net/dm_vincent/article/details/41693125
- http://www.rendoumi.com/elasticsearchzhong-de/