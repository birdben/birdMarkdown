---
title: "ElasticSearch官网翻译_FieldDatatypes"
date: 2017-05-10 23:31:42
tags: [Elasticsearch]
categories: [Search]
---

#### Field datatypes

Elasticsearch支持文档中字段的一些不同的数据类型：

#### Core datatypes

- String datatype
 * string
- Numeric datatypes
 * long, integer, short, byte, double, float
- Date datatype
 * date
- Boolean datatype
 * boolean
- Binary datatype
 * binary

 
#### Complex datatypes

- Array datatype
 * Array support does not require a dedicated type
- Object datatype
 * object for single JSON objects
- Nested datatype
 * nested for arrays of JSON objects

#### Geo datatypes

- Geo-point datatype
 * geo_point for lat/lon points
- Geo-Shape datatype
 * geo_shape for complex shapes like polygons

#### Specialised datatypes

- IPv4 datatype
 * ip for IPv4 addresses
- Completion datatype
 * completion to provide auto-complete suggestions
- Token count datatype
 * token_count to count the number of tokens in a string
- mapper-murmur3
 * murmur3 to compute hashes of values at index-time and store them in the index
- Attachment datatype
 * See the mapper-attachments plugin which supports indexing attachments like Microsoft Office formats, Open Document formats, ePub, HTML, etc. into an attachment datatype

#### Multi-fields

为了不同的目的，以不同的方式索引相同的字段通常是有用的。 例如，字符串字段可以作为全文搜索的analyzed字段进行索引，并且作为用于排序或聚合的not_analyzed字段。 或者，你可以使用standard分析器，english和french分析器索引字符串字段。

这是multi-fields的目的。 大多数数据类型通过fields参数支持multi-fields。

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/2.4/mapping-types.html
