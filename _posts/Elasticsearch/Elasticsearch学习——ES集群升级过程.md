---
title: "Elasticsearch学习——ES集群升级过程"
date: 2017-07-07 12:27:33
tags: [Elasticsearch]
categories: [Search]
---

## ES集群从2.3版本升级到5.3版本

升级的过程中还是建议全集群停机后升级，升级前一定要进行数据备份（强烈建议），防止升级后索引文件损坏，数据无法恢复。

#### 1. 停止Logstash写入

#### 2. 关闭ES的自动分片

elasticsearch可以通过reroute API来手动进行索引分片的分配。不过要想完全手动，必须先把cluster.routing.allocation.enable参数设置为none，这个参数默认是开启的，默认情况下当ES节点实例启动时，会尝试从其他节点实例上拷贝相关的shard副本至本地，这样会浪费大量的时间和耗费高额的IO资源。禁止ES进行自动索引分片分配，这样做的目的在于当集群shutdown之后可以快速的启动。如果关闭该选项，那么当新的ES节点实例启动，尝试加入集群的时候，它不会从其他实例上拷贝shard副本。当ES节点实例完全启动之后，则应该再将该选项开启，以提供长期的容灾。尤其当数据量很大的时候，这个参数必须要设置，因为如果不设置，在我们重启集群的时候，ES会自动进行分片的迁移，导致大量资源被占用，重启变慢。

```
$ curl -XPUT http://127.0.0.1:9200/_cluster/settings -d '{
  "transient" : {
    "cluster.routing.allocation.enable" : "none"
  }
}'
```

#### 3. 同步flush操作

如果不允许任何丢失，需要执行该操作，如果可以容忍在短时间内的数据丢失，可以忽略这一步骤。手动运行一次synced flush，同步副本分片的 commit id，缩小恢复时的网络传输带宽。将内存数据同步到磁盘(可选)

```
$ curl -XPOST http://127.0.0.1:9200/_flush/synced
```

#### 4. 添加path.repo，创建snapshot备份索引数据

需要在elasticsearch.yml配置文件中，配置path.repo目录，然后创建Repository如下：

```
$ curl -XPOST 'http://localhost:9200/_snapshot/my_backup?pretty' -d '{
    "type": "fs", 
    "settings": {
        "location": "/data0/es_logs",
        "compress": true
    }
}'
```

备份名字是nginx_access_logs_index_前缀开始的索引数据到nginx_access_logs_index_snapshot

```
# 备份snapshot
$ curl -XPOST 'http://localhost:9200/_snapshot/my_backup/nginx_access_logs_index_snapshot?pretty' -d '{
    "indices": "nginx_access_logs_index_*"
}'

# 删除snapshot
$ curl -XDELETE 'http://localhost:9200/_snapshot/my_backup/nginx_access_logs_index_snapshot?pretty'

# 查看snapshot备份状态
$ curl -XGET 'http://localhost:9200/_snapshot/my_backup/nginx_access_logs_index_snapshot/_status?pretty'

# 查看所有snapshot
$ curl -XGET 'http://localhost:9200/_snapshot/my_backup/_all?pretty'

# 查看当前snapshot
$ curl -XGET 'http://localhost:9200/_snapshot/my_backup/_current?pretty'
```

如果需要恢复snapshot

```
# 恢复snapshot
$ curl -XPOST 'http://localhost:9200/_snapshot/my_backup/nginx_access_logs_index_snapshot/_restore?pretty'

# 查看snapshot恢复状态
$ curl -XGET 'http://localhost:9200/_snapshot/my_backup/nginx_access_logs_index_snapshot/_status?pretty'
```

#### 5. 使用elasticsearch-migration插件检查升级的注意事项

安装elasticsearch-migration插件

```
$ ./bin/plugin install https://github.com/elastic/elasticsearch-migration/releases/download/v2.0.4/elasticsearch-migration-2.0.4.zip
```

安装成功之后访问：http://localhost:9200/_plugin/elasticsearch-migration

![Migration Helper](http://img.blog.csdn.net/20170705180327260?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvYmlyZGJlbg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

点击Cluster Checkup进行检查

![插件检查](http://img.blog.csdn.net/20170707110835946?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvYmlyZGJlbg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

- 插件检查

提示bigdesk，head，kopf插件已经不再被支持

![节点设置](http://img.blog.csdn.net/20170705180227995?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvYmlyZGJlbg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

- 节点设置

提示某些设置已经被修改

需要修改/etc/sysctl.conf配置文件，然后执行"sysctl -p"。

```
vm.max_map_count=262144
```

![索引](http://img.blog.csdn.net/20170705180150980?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvYmlyZGJlbg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

- 索引

提示按照IP字段聚合不再返回数字类型的值，这里对于我们来说没有什么影响，因为我们并没有按照IP字段进行聚合操作。

#### 6. 迁移ELK的配置文件到Docker环境

- 将Logstash替换成Docker容器运行
- 将Elasticsearch替换成Docker容器运行
- 将Kibana替换成Docker容器运行

#### 7. 线上ES全集群统一停止进程

需要同时停止所有集群的es服务，然后挨个重启。如果是rolling upgrade，则需要关闭所要升级版本的节点实例，并将其移除集群。我这里是全集群停止进程。

```
$ curl -XPOST http://localhost:9200/_cluster/nodes/_local/_shutdown
```

#### 8. 插件升级更新

bigdesk，head，kopf等插件暂时不做升级了，ik插件因为时间的原因暂时没有做升级，后续会补充上。

#### 9. 启动ES的Docker容器

注意设置ES的内存大小。

#### 10. 修改索引模板

ES 5.x中String字段类型已经被拆分为Text字段类型和Keyword字段类型了，所以对应的索引模板需要做相应的修改。并且在ES 5.x环境创建新的索引模板（需要注意elasticsearch-migration中提示的重大变化）。我这里因为之前部分索引使用ik插件，所以必须重新索引数据。

#### 11. 开启自动分片

等待各节点都加入到集群以后，恢复自动分片分配。

```
$ curl -XPUT http://127.0.0.1:9200/_cluster/settings -d '{
  "transient" : {
    "cluster.routing.allocation.enable" : "all"
  }
}'
```

#### 12. 启动Logstash的Docker容器

使用Logstash测试配置（新的Consumer Group读取Kafka数据）写入数据到ES 5.x环境。索引数据测试通过之后，就可以使用线上Logstash的配置启动Docker容器了。

### 问题总结

#### ik插件无法使用原来映射配置，需要重新索引数据

之前ES 2.x版本中使用了ik插件作为中文分析器，但是ES 5.x版本的ik插件的配置有些变化，无法直接使用原来的索引数据，直接使用ES 5.x版本启动则会报错无法识别映射中的ik配置，所以需要重新索引数据。

- 注意：ik插件5.0.0版本

移除名为 ik 的analyzer和tokenizer,请分别使用 ik_smart 和 ik_max_word

```
[2017-07-06 15:25:49,511][WARN ][cluster.action.shard     ] [Diablo] [java_logs_index_2017.06.08][2] received shard failed for target shard [[java_logs_index_2017.06.08][2], node[IpN6gRx8RMOThPXzhPCWJQ], [P], v[53], restoring[my_backup:java_logs_index_snapshot], s[INITIALIZING], a[id=9wDgtC_2QVqmjROGJMUgow], unassigned_info[[reason=ALLOCATION_FAILED], at[2017-07-06T15:25:48.038Z], details[failed to update mappings, failure MapperParsingException[analyzer [ik] not found for field [msg]]]]], indexUUID [qjDk1AG_RvW1cJUXOCl4bw], message [failed to update mappings], failure [MapperParsingException[analyzer [ik] not found for field [msg]]]
MapperParsingException[analyzer [ik] not found for field [msg]]
	at org.elasticsearch.index.mapper.core.TypeParsers.parseAnalyzersAndTermVectors(TypeParsers.java:213)
	at org.elasticsearch.index.mapper.core.TypeParsers.parseTextField(TypeParsers.java:250)
	at org.elasticsearch.index.mapper.core.StringFieldMapper$TypeParser.parse(StringFieldMapper.java:170)
	at org.elasticsearch.index.mapper.object.ObjectMapper$TypeParser.parseProperties(ObjectMapper.java:309)
	at org.elasticsearch.index.mapper.object.ObjectMapper$TypeParser.parseObjectOrDocumentTypeProperties(ObjectMapper.java:222)
	at org.elasticsearch.index.mapper.object.RootObjectMapper$TypeParser.parse(RootObjectMapper.java:139)
	at org.elasticsearch.index.mapper.DocumentMapperParser.parse(DocumentMapperParser.java:118)
	at org.elasticsearch.index.mapper.DocumentMapperParser.parse(DocumentMapperParser.java:99)
	at org.elasticsearch.index.mapper.MapperService.parse(MapperService.java:549)
	at org.elasticsearch.index.mapper.MapperService.merge(MapperService.java:319)
	at org.elasticsearch.indices.cluster.IndicesClusterStateService.processMapping(IndicesClusterStateService.java:410)
	at org.elasticsearch.indices.cluster.IndicesClusterStateService.applyMappings(IndicesClusterStateService.java:371)
	at org.elasticsearch.indices.cluster.IndicesClusterStateService.clusterChanged(IndicesClusterStateService.java:175)
	at org.elasticsearch.cluster.service.InternalClusterService.runTasksForExecutor(InternalClusterService.java:622)
	at org.elasticsearch.cluster.service.InternalClusterService$UpdateTask.run(InternalClusterService.java:784)
	at org.elasticsearch.common.util.concurrent.PrioritizedEsThreadPoolExecutor$TieBreakingPrioritizedRunnable.runAndClean(PrioritizedEsThreadPoolExecutor.java:231)
	at org.elasticsearch.common.util.concurrent.PrioritizedEsThreadPoolExecutor$TieBreakingPrioritizedRunnable.run(PrioritizedEsThreadPoolExecutor.java:194)
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1145)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:615)
	at java.lang.Thread.run(Thread.java:745)
```

#### Logstash启动报错

这个问题也挺奇葩的，之前Logstash都没什么问题，突然启动就crash了。后来在网上找到了原因，这个是Logstash的一个bug，不能在patterns目录下存在任何目录，否则Logstash启动就会crash。

可以参考：https://github.com/logstash-plugins/logstash-filter-grok/issues/110

```
Settings: Default pipeline workers: 4
Pipeline aborted due to error {:exception=>"NoMethodError", :error=>"undefined method `close' for nil:NilClass", :backtrace=>["/opt/logstash/vendor/bundle/jruby/1.9/gems/jls-grok-0.11.4/lib/grok-pure.rb:83:in `add_patterns_from_file'", "/opt/logstash/vendor/bundle/jruby/1.9/gems/logstash-filter-grok-2.0.5/lib/logstash/filters/grok.rb:372:in `add_patterns_from_files'", "org/jruby/RubyArray.java:1613:in `each'", "/opt/logstash/vendor/bundle/jruby/1.9/gems/logstash-filter-grok-2.0.5/lib/logstash/filters/grok.rb:368:in `add_patterns_from_files'", "/opt/logstash/vendor/bundle/jruby/1.9/gems/logstash-filter-grok-2.0.5/lib/logstash/filters/grok.rb:263:in `register'", "org/jruby/RubyArray.java:1613:in `each'", "/opt/logstash/vendor/bundle/jruby/1.9/gems/logstash-filter-grok-2.0.5/lib/logstash/filters/grok.rb:259:in `register'", "org/jruby/RubyHash.java:1342:in `each'", "/opt/logstash/vendor/bundle/jruby/1.9/gems/logstash-filter-grok-2.0.5/lib/logstash/filters/grok.rb:255:in `register'", "/opt/logstash/vendor/bundle/jruby/1.9/gems/logstash-core-2.4.1-java/lib/logstash/pipeline.rb:182:in `start_workers'", "org/jruby/RubyArray.java:1613:in `each'", "/opt/logstash/vendor/bundle/jruby/1.9/gems/logstash-core-2.4.1-java/lib/logstash/pipeline.rb:182:in `start_workers'", "/opt/logstash/vendor/bundle/jruby/1.9/gems/logstash-core-2.4.1-java/lib/logstash/pipeline.rb:136:in `run'", "/opt/logstash/vendor/bundle/jruby/1.9/gems/logstash-core-2.4.1-java/lib/logstash/agent.rb:491:in `start_pipeline'"], :level=>:error}
stopping pipeline {:id=>"main"}
```

#### docker容器时区不正确，导致logstash转换@timestamp时间不正确

升级之前的索引时间和升级之后的索引时间不一致

```
# 宿主机时间和时区
$ date -R
Thu, 06 Jul 2017 18:00:03 +0800
```

```
# docker容器时间和时区
$ date -R
Thu, 06 Jul 2017 10:00:44 +0000
```

需要在Logstash的Dockerfile中添加如下的配置来修改时区

```
RUN echo "Asia/Shanghai" > /etc/timezone
RUN ln -sf /usr/share/zoneinfo/Asia/Shanghai  /etc/localtime
```

参考文章：

- https://github.com/elastic/elasticsearch-migration
- https://github.com/medcl/elasticsearch-analysis-ik
- https://github.com/chenryn/ELKstack-guide-cn/blob/master/elasticsearch/upgrade.md
- https://www.elastic.co/guide/en/elasticsearch/reference/5.x/restart-upgrade.html
- https://www.elastic.co/guide/en/elasticsearch/reference/5.x/rolling-upgrades.html
- https://www.elastic.co/guide/en/elasticsearch/reference/5.x/setup-upgrade.html
- http://suclogger.com/线上Elasticsearch集群升级到5-X版本/
- https://github.com/logstash-plugins/logstash-filter-grok/issues/110
