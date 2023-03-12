---
title: "ElasticSearch官网翻译_FullTextQueries"
date: 2017-06-13 11:56:56
tags: [Elasticsearch]
categories: [Search]
---

## Full text queries

高级全文查询通常用于在全文本字段（如电子邮件正文）上运行全文查询。他们了解如何对被查询的字段进行分析，并在执行前将每个字段的analyzer（或search_analyzer）应用于查询字符串。

该组中的查询是：

- match query（全文匹配查询）

用于执行全文查询的标准查询，包括fuzzy（模糊）匹配和phrase（短语）或proximity（邻近）查询。

- match_phrase query（短语匹配查询）

像match query一样，但用于匹配确切的短语或单词接近度匹配。

- match_phrase_prefix query（短语前缀匹配查询）

一种弱类型的查询。 像match_phrase查询一样，但是可以对有决定性的单词进行wildcard（通配符）搜索。

- multi_match query（多字段匹配查询）

multi-field版本的match query。

- common_terms query（常用术语查询）

可以对一些比较专业的偏门词语进行更加专业的查询。

- query_string query（查询字符串查询）

支持严谨的Lucene query string syntax（查询字符串语法），允许你在单个查询字符串中指定AND | OR | NOT条件和multi-field search（多字段搜索）。 仅适用于专家用户。

- simple_query_string（简单查询字符串查询）

一个更简单，更健壮的query_string语法版本，适合直接暴露给用户。

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/5.4/full-text-queries.html
