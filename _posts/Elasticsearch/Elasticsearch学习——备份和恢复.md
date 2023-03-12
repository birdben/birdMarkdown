---
title: "Elasticsearch学习——备份和恢复"
date: 2017-07-09 17:46:02
tags: [Elasticsearch]
categories: [Search]
---

基本步骤如下：

- 添加repo目录配置
- 创建快照仓储Repository
- 备份snapshot
- 恢复snapshot
- 重新索引数据

### 添加repo目录配置

在elasticsearch.yml配置文件中添加repo目录配置

```
path.repo: /data0/es_logs
```

### 创建快照仓储Repository

需要创建一个repository，location需要指定为path.repo配置对应目录。

```
$ curl -XPOST 'http://localhost:9200/_snapshot/my_backup?pretty' -d '{
    "type": "fs", 
    "settings": {
        "location": "/data0/es_logs",
        "compress": true
    }
}'
```

### 备份snapshot

备份snapshot时，需要指定备份的repository和新创建的snapshot名称，indices参数是备份的索引名称（支持*通配符）。

```
# 在指定repository下创建指定名称的snapshot（指定备份的索引）
$ curl -XPOST 'http://localhost:9200/_snapshot/my_backup/nginx_access_logs_index_snapshot?pretty' -d '{
    "indices": "nginx_access_logs_index_*"
}'

# 在指定repository下删除指定名称的snapshot
$ curl -XDELETE 'http://localhost:9200/_snapshot/my_backup/nginx_access_logs_index_snapshot?pretty'

# 在指定repository下查看指定名称的snapshot状态
$ curl -XGET 'http://localhost:9200/_snapshot/my_backup/nginx_access_logs_index_snapshot/_status?pretty'

# 在指定repository下查看所有的snapshot
$ curl -XGET 'http://localhost:9200/_snapshot/my_backup/_all?pretty'

# 在指定repository下查看正在进行的snapshot
$ curl -XGET 'http://localhost:9200/_snapshot/my_backup/_current?pretty'
```

备份成功后，会在配置的path.repo目录下生成如下几个文件：

![snapshot_create_files](http://img.blog.csdn.net/20170709145920288?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvYmlyZGJlbg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

- meta开头的文件应该是记录ID为IR6HjdgeR1aqHZJMjP3WVA的snapshot元数据信息
- snap开头的文件应该是记录ID为IR6HjdgeR1aqHZJMjP3WVA的snapshot快照信息
- indices目录下是具体的索引备份数据
- index.latest文件应该是记录的最新的版本号
- index-0文件内容是snapshot和对应snapshot的index信息，具体内容如下：

```
{
  "snapshots": [
    {
      "name": "nginx_access_logs_index_snapshot",
      "uuid": "IR6HjdgeR1aqHZJMjP3WVA"
    }
  ],
  "indices": {
    "nginx_access_logs_index_2017.06": {
      "id": "Qi4ieawGQBG1bb_-HbeADw",
      "snapshots": [
        {
          "name": "nginx_access_logs_index_snapshot",
          "uuid": "IR6HjdgeR1aqHZJMjP3WVA"
        }
      ]
    }
  }
}
```

当删除nginx_access_logs_index_snapshot快照后，ID为IR6HjdgeR1aqHZJMjP3WVA的snapshot的meta和snap文件被删除了，indices目录下的数据文件被删除了，生成了新的index-1文件，index-0文件内容没有变化，path.repo目录下文件如下：

![snapshot_delete_files](http://img.blog.csdn.net/20170709151531652?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvYmlyZGJlbg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

index-1文件内容如下：

```
{"snapshots":[],"indices":{}}
```

### 恢复snapshot

如果要恢复其他ES集群备份的snapshot，同样需要在要恢复数据的ES节点先配置path.repo，并且创建快照仓储Repository，然后我们可以将上面备份snapshot生成的几个文件（index-0，indices，meta-ID，snap-ID）复制到该ES节点的path.repo目录下，然后查看指定repository下的所有的snapshot，就可以看到我们复制过来的snapshot。

```
curl -XGET 'http://localhost:9200/_snapshot/my_backup/_all?pretty'
{
  "snapshots" : [
    {
      "snapshot" : "nginx_access_logs_index_snapshot",
      "uuid" : "IR6HjdgeR1aqHZJMjP3WVA",
      "version_id" : 5030199,
      "version" : "5.3.1",
      "indices" : [
        "nginx_access_logs_index_2017.06"
      ],
      "state" : "SUCCESS",
      "start_time" : "2017-06-23T04:30:07.853Z",
      "start_time_in_millis" : 1498192207853,
      "end_time" : "2017-06-23T04:30:25.384Z",
      "end_time_in_millis" : 1498192225384,
      "duration_in_millis" : 17531,
      "failures" : [ ],
      "shards" : {
        "total" : 4,
        "failed" : 0,
        "successful" : 4
      }
    }
  ]
}
```

然后可以通过_restore指定恢复repository下的snapshot

```
# 关闭索引（如果索引已经存在目标的集群，需要先关闭索引，恢复数据后再打开）
$ curl -XPOST 'http://localhost:9200/nginx_access_logs_index_*/_close?pretty'

# 在指定repository下恢复指定名称的snapshot
$ curl -XPOST 'http://localhost:9200/_snapshot/my_backup/nginx_access_logs_index_snapshot/_restore?pretty'

# 打开索引（如果索引已经存在目标的集群，需要先关闭索引，恢复数据后再打开）
$ curl -XPOST 'http://localhost:9200/nginx_access_logs_index_*/_open?pretty'

# 在指定repository下查看指定名称的snapshot状态
$ curl -XGET 'http://localhost:9200/_snapshot/my_backup/nginx_access_logs_index_snapshot/_status?pretty'
```

### 重新索引数据

如果有需要也可以进行重新索引数据。

```
$ curl -XPOST 'http://localhost:9200/_reindex?pretty' -d '{
  "source": {
    "index": "nginx_access_logs_index_2017.06",
    "query": {
      "range": {
        "@timestamp": {
          "lt" : 1497860825147
        }
      }
    }
  },
  "dest": {
    "index": "v1_nginx_access_logs_index_2017.06"
  }
}'
```


参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/5.4/modules-snapshots.html
- https://www.elastic.co/guide/en/elasticsearch/reference/5.4/docs-reindex.html
