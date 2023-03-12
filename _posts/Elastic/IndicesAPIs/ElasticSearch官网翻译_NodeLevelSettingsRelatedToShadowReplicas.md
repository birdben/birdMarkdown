---
title: "ElasticSearch官网翻译_NodeLevelSettingsRelatedToShadowReplicas"
date: 2017-06-05 16:58:35
tags: [Elasticsearch]
categories: [Search]
---

## Node level settings related to shadow replicas

这些是需要在elasticsearch.yml中配置的非动态设置

- node.add_lock_id_to_custom_path

布尔值设置，指示Elasticsearch是否应将节点的序数附加到自定义数据路径。 例如，如果启用了此功能并使用了"/tmp/foo"路径，则第一个本地运行的节点将使用"/tmp/foo/0"，第二个将使用"/tmp/foo/1"，第三个"/tmp/foo/2"以此类推。默认为true。

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/5.4/_node_level_settings_related_to_shadow_replicas.html
