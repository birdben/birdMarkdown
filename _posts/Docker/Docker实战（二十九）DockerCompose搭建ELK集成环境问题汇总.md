---
title: "Docker实战（二十九）DockerCompose搭建ELK集成环境问题汇总"
date: 2017-05-06 18:17:17
tags: [Docker环境]
categories: [Docker]
---

今天记录一下在使用docker-compose构建ELK集成环境时遇到的坑，废话不多说了直接来踩坑。

### docker网络冲突

在修改好ELK的docker-compose.yml配置文件后，尝试启动遇到网络冲突的问题，错误提示说"172.18.0.1"这个网络已经存在。

```
$ docker-compose up -d
Creating network "5x_elk_net" with driver "bridge"
ERROR: failed to allocate gateway (172.18.0.1): Address already in use

# 查看现在的docker网络，发现下面几个已经存在的网络配置
$ docker network ls

NETWORK ID          NAME                      DRIVER              SCOPE
27822b9fb5c5        bridge                    bridge              local
08b6d63e27d2        host                      host                local
dd0874e0e097        none                      null                local
bacc9a64bb83        test_default              bridge              local

# 依次查看这几个网络的配置，发现bacc9a64bb83这个容器的网络已经使用了"172.18.0.1"
$ docker network inspect bacc9a64bb83

[
    {
        "Name": "test_default",
        "Id": "bacc9a64bb8323b2e53b1c85b4643061d38699227492f9174855202b6900252a",
        "Created": "2017-04-21T10:29:37.26843596Z",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": null,
            "Config": [
                {
                    "Subnet": "172.18.0.0/16",
                    "Gateway": "172.18.0.1"
                }
            ]
        },
        "Internal": false,
        "Attachable": false,
        "Containers": {},
        "Options": {},
        "Labels": {}
    }
]
```

找到冲突的地方就好办，两种方式来解决：

1. 删除已经存在的网络
2. 更换docker-compose现有的网段

因为这个容器对我还有其他用处，所以这里我选择更换docker-compose的网络来解决


### logstash-output-elasticsearch插件的host配置不支持特殊符号

下面是我的docker-compose.yml配置文件（篇幅原因，这里省略了一部分，只是用了ES的容器配置举例）。

docker-compose.yml配置文件链接：

- https://github.com/birdben/birdDocker/blob/v2/elk/5.x/docker-compose.yml

```
version: '2'
services:

   ...

   elasticsearch:
      # 指定当前构建的Docker容器的镜像
      image: birdben/elasticsearch_5.x:v2
      restart: always
      # 指定当前构建的Docker容器的名称
      container_name: elasticsearch_5.x
      networks:
         elk_net:
            # 指定当前构建的Docker容器的IP地址
            ipv4_address: 172.20.0.5
      # 指定当前构建的Docker容器的host配置
      extra_hosts:
         - "filebeat:172.20.0.2"
         - "redis:172.20.0.3"
         - "logstash:172.20.0.4"
         - "elasticsearch:172.20.0.5"
         - "kibana:172.20.0.6"
      # 指定当前构建的Docker容器的volume挂在目录设置
      volumes:
         - /Users/yunyu/workspace_git/birdDocker/elk/5.x/volumes/elasticsearch/data:/usr/share/elasticsearch/data
         - /Users/yunyu/workspace_git/birdDocker/elk/5.x/volumes/elasticsearch/config:/usr/share/elasticsearch/config
         - /Users/yunyu/workspace_git/birdDocker/elk/5.x/volumes/elasticsearch/logs:/usr/share/elasticsearch/logs
      # 指定当前构建的Docker容器对外开放的端口号映射
      ports:
         - "9200:9200"
         - "9300:9300"

  ...

```

如果ES容器的container_name配置为"elasticsearch_5.x"，那么logstash需要在logstash.conf配置文件中使用host来指定ES的服务器为"elasticsearch_5.x"，配置如下：

```
...

output {
    stdout {
        codec => rubydebug
    }
    elasticsearch {
        codec => "json"
        hosts => ["elasticsearch_5.x:9200"]
        index => "logstash-%{+YYYY.MM.dd}"
        document_type => "%{type}"
        workers => 1
        flush_size => 20000
        idle_flush_time => 10
    }
}
```

启动Logstash之后，会如下报错

```
Sending Logstash's logs to /usr/share/logstash/logs which is now configured via log4j2.properties
[2017-05-06T11:09:25,349][INFO ][logstash.setting.writabledirectory] Creating directory {:setting=>"path.queue", :path=>"/usr/share/logstash/data/queue"}
[2017-05-06T11:09:25,382][INFO ][logstash.agent           ] No persistent UUID file found. Generating new UUID {:uuid=>"e241d497-58e2-46de-9213-95088242255a", :path=>"/usr/share/logstash/data/uuid"}
[2017-05-06T11:09:25,771][ERROR][logstash.agent           ] Cannot load an invalid configuration {:reason=>"bad URI(is not URI?): elasticsearch_5.x:9200"}
```

提示是"elasticsearch_5.x"是一个非法的URI地址，将docker-compose.yml配置文件的container_name和logstash.conf的host配置修改为"elasticsearch5x"之后就不会再报错了。所以推断logstash-output-elasticsearch插件对host要求比较严格，不支持一些特殊符号。

### docker-compose配置的容器无法全部正常启动

使用docker-compose启动ELK的2.x版本服务都一切正常，但是换成ELK的5.x版本后，发现Filebeat，Logstash，Redis，Kibana服务都正常，只有ES的容器起来没有多久就自己挂掉了。
看了ES的日志也没有发现什么异常，单独启动ES5.x的容器却能正常使用。

后来实在没办法了，我尝试在docker-compose.yml配置文件中只留下ES的容器，这样运行也没问题。之后尝试一个一个将其他容器的配置加到docker-compose.yml配置文件，发现当Logstash5.x和ES5.x的容器同时启动，ES的容器就会出现上面自己挂掉的情况。

然后我又仔细查看了一下ES的日志文件，发现了一些区别：有问题的ES日志中多了一些GC的日志。

- 有问题的ES日志

```
[2017-05-06T10:36:30,804][INFO ][o.e.n.Node               ] [node-1] initialized
[2017-05-06T10:36:30,805][INFO ][o.e.n.Node               ] [node-1] starting ...
[2017-05-06T10:36:31,162][WARN ][i.n.u.i.MacAddressUtil   ] Failed to find a usable hardware address from the network interfaces; using random bytes: 37:86:d0:ae:ee:3d:71:88
[2017-05-06T10:36:31,437][INFO ][o.e.t.TransportService   ] [node-1] publish_address {172.20.0.5:9300}, bound_addresses {[::]:9300}
[2017-05-06T10:36:31,457][INFO ][o.e.b.BootstrapChecks    ] [node-1] bound or publishing to a non-loopback or non-link-local address, enforcing bootstrap checks
[2017-05-06T10:36:33,545][WARN ][o.e.m.j.JvmGcMonitorService] [node-1] [gc][young][2][2] duration [1.4s], collections [1]/[1.6s], total [1.4s]/[1.7s], memory [284.7mb]->[51.7mb]/[1.9gb], all_pools {[young] [266.2mb]->[11.2mb]/[266.2mb]}{[survivor] [18.4mb]->[32mb]/[33.2mb]}{[old] [0b]->[8.4mb]/[1.6gb]}
[2017-05-06T10:36:33,560][WARN ][o.e.m.j.JvmGcMonitorService] [node-1] [gc][2] overhead, spent [1.4s] collecting in the last [1.6s]
[2017-05-06T10:36:50,112][INFO ][o.e.n.Node               ] [node-1] initializing ...
[2017-05-06T10:36:50,400][INFO ][o.e.e.NodeEnvironment    ] [node-1] using [1] data paths, mounts [[/usr/share/elasticsearch/data (osxfs)]], net usable_space [6.1gb], net total_space [232.6gb], spins? [possibly], types [fuse.osxfs]
[2017-05-06T10:36:50,401][INFO ][o.e.e.NodeEnvironment    ] [node-1] heap size [1.9gb], compressed ordinary object pointers [true]
[2017-05-06T10:36:50,415][INFO ][o.e.n.Node               ] [node-1] node name [node-1], node ID [x7vSjbIKSdeUbHcAjXWPCw]
[2017-05-06T10:36:50,417][INFO ][o.e.n.Node               ] [node-1] version[5.3.1], pid[1], build[5f9cf58/2017-04-17T15:52:53.846Z], OS[Linux/4.9.13-moby/amd64], JVM[Oracle Corporation/OpenJDK 64-Bit Server VM/1.8.0_121/25.121-b13]
```

- 没有问题的ES日志

```
[2017-05-06T09:35:52,233][INFO ][o.e.n.Node               ] [node-1] initialized
[2017-05-06T09:35:52,238][INFO ][o.e.n.Node               ] [node-1] starting ...
[2017-05-06T09:35:52,408][WARN ][i.n.u.i.MacAddressUtil   ] Failed to find a usable hardware address from the network interfaces; using random bytes: 4a:ab:e0:6f:82:87:b0:e5
[2017-05-06T09:35:52,569][INFO ][o.e.t.TransportService   ] [node-1] publish_address {172.20.0.5:9300}, bound_addresses {[::]:9300}
[2017-05-06T09:35:52,592][INFO ][o.e.b.BootstrapChecks    ] [node-1] bound or publishing to a non-loopback or non-link-local address, enforcing bootstrap checks
[2017-05-06T09:35:55,713][INFO ][o.e.c.s.ClusterService   ] [node-1] new_master {node-1}{Z0Yoi2zfTl237aiVzEoOug}{N4z8452FTc-SArP7hh7h-g}{172.20.0.5}{172.20.0.5:9300}, reason: zen-disco-elected-as-master ([0] nodes joined)
[2017-05-06T09:35:55,779][INFO ][o.e.g.GatewayService     ] [node-1] recovered [0] indices into cluster_state
[2017-05-06T09:35:55,790][INFO ][o.e.h.n.Netty4HttpServerTransport] [node-1] publish_address {172.20.0.5:9200}, bound_addresses {[::]:9200}
[2017-05-06T09:35:55,822][INFO ][o.e.n.Node               ] [node-1] started
[2017-05-06T09:35:58,039][INFO ][o.e.c.m.MetaDataCreateIndexService] [node-1] [logstash-2017.05.06] creating index, cause [auto(bulk api)], templates [logstash], shards [5]/[1], mappings [_default_]
[2017-05-06T09:35:58,663][INFO ][o.e.c.m.MetaDataMappingService] [node-1] [logstash-2017.05.06/h2c9vcE2TaCdXgYxSjh0IA] create_mapping [log]
[2017-05-06T09:36:01,108][INFO ][o.e.c.m.MetaDataCreateIndexService] [node-1] [.kibana] creating index, cause [api], templates [], shards [1]/[1], mappings [server, config]
[2017-05-06T09:36:25,753][WARN ][o.e.c.r.a.DiskThresholdMonitor] [node-1] high disk watermark [90%] exceeded on [Z0Yoi2zfTl237aiVzEoOug][node-1][/usr/share/elasticsearch/data/nodes/0] free: 6.1gb[2.6%], shards will be relocated away from this node
```

经过上面的分析，我怀疑是我docker服务设置的内存大小无法支持我启动这么多的容器。后来发现ES和Logstash的5.x版本比2.x版本多了一个jvm.options的配置文件，主要是用来设置ES和Logstash的JVM的配置使用的，在这个配置文件里可以控制JVM的堆大小。这里将ES和Logstash的堆内存调小后，再使用docker-compose启动，ES5.x的容器已经能够正常启动了。

ES的jvm.options

```
-Xms2g
-Xmx2g

# 修改为

-Xms1g
-Xmx1g
```

Logstash的jvm.options

```
-Xms256m
-Xmx1g

# 修改为

-Xms256m
-Xmx256m
```

参考文章：

- http://www.open-open.com/lib/view/open1451606865542.html
- https://github.com/logstash-plugins/logstash-output-elasticsearch/issues/400