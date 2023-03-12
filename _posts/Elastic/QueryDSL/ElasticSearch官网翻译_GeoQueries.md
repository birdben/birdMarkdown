---
title: "ElasticSearch官网翻译_GeoQueries"
date: 2017-06-30 18:23:42
tags: [Elasticsearch]
categories: [Search]
---

## Geo queries

Elasticsearch支持两种类型的geo数据：支持lat/lon对的geo_point字段和支持points（点），lines（线），circles（圆），polygons（多边形），multi-polygons（多个多边形）等的geo_shape字段。

该组中的查询是：

- geo_shape query（地理形状查询）

查找与指定的geo-shape相交，包含，或不相交的geo-shapes（地理形状）的文档。

- geo_bounding_box query（地理位置边界查询）

查找在指定矩形中的geo-points（地理点）的文档。

- geo_distance query（地理距离查询）

查找在中心点的指定距离内的geo-points（地理点）的文档。

- geo_distance_range query（地理距离范围查询）

像geo_point查询一样，但范围从中心点开始指定的距离。

- geo_polygon query（地理多边形查询）

查找在指定多边形内的geo-points（地理点）的文档。

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/5.4/geo-queries.html
