---
title: "Docker实战（三十四）Docker安装Redis集群环境"
date: 2017-07-29 16:28:00
tags: [Docker命令, Dockerfile, Redis]
categories: [Docker]
---

### Redis集群环境

至少需要3(Master) + 3(Slave)才能建立集群，是无中心的分布式存储架构，这里我使用docker-compose部署了3个master和3个slave节点。

Name|IP|Inner Port|Outter Port
---|---|---|---
redism1|172.22.0.2|6379|6379
redism2|172.22.0.3|6379|6479
redism3|172.22.0.4|6379|6579
rediss1|172.22.0.5|6379|6679
rediss2|172.22.0.6|6379|6779
rediss3|172.22.0.7|6379|6879

#### redis.conf配置文件

```
# 不能开启后台运行，否则Docker容器无法运行
daemonize no
# 开启集群模式
cluster-enabled yes
# 集群配置文件，我这里每个节点的配置文件名字不同
cluster-config-file redism1.conf
# 集群节点超时时间
cluster-node-timeout 15000
```

每个redis节点的配置文件都按照上面的方式配置。

### 使用redis-trib.rb创建Redis集群

将redis源代码的src目录下的集群管理程序redis-trib.rb复制到/usr/bin目录，并将bin目录加入到环境变量PATH中，以简化后续的操作。

```
# 先将redis-trib.rb脚本复制到/usr/bin目录下
$ cp /redis/config/redis-trib.rb /usr/bin/
```

> 注意：
> 
> 要使用redis-trib.rb需要安装ruby环境

#### 安装ruby

```
curl -sSL https://get.rvm.io | bash -s stable
echo "export PATH=$PATH:/usr/local/rvm/bin" >> ~/.bashrc
source ~/.bashrc
rvm install ruby-2.3.1
```

#### 安装gem

```
apt-get install gem
apt-get install rubygems
gem install redis
```

> 注意：
> 
> 这里使用Docker部署安装Redis集群有个问题，Redis集群目前不支持announce-ip和announce-port，所以redis集群在Docker外部的网络无法互相发现，github上已经有相关issue应该会在4.0版本fix
> 
> 具体issue地址：https://github.com/antirez/redis/issues/2527
> 
> 这里我为了自己搭建环境简单，我使用docker-compose创建了一个redis集群的网络，然后单独创建了一个redis-trib的容器，用来在Docker内部的网络单独创建redis集群，这样就不会出现无法通信的问题，但是Docker外部的客户端仍然无法连接到redis集群

#### 创建Redis集群

这里是在单独创建的redis-trib容器中执行的创建redis集群的命令

```
# 创建redis集群（前3个是master节点，后3个是slave节点）
$ redis-trib.rb create --replicas 1 172.22.0.2:6379 172.22.0.3:6379 172.22.0.4:6379 172.22.0.5:6379 172.22.0.6:6379 172.22.0.7:6379
>>> Creating cluster
>>> Performing hash slots allocation on 6 nodes...
Using 3 masters:
172.22.0.2:6379
172.22.0.3:6379
172.22.0.4:6379
Adding replica 172.22.0.5:6379 to 172.22.0.2:6379
Adding replica 172.22.0.6:6379 to 172.22.0.3:6379
Adding replica 172.22.0.7:6379 to 172.22.0.4:6379
M: ec4533266ac9cc2365953e39429f32bf42c64484 172.22.0.2:6379
   slots:0-5460 (5461 slots) master
M: e9eab3f50bebc2076f0efd453b5d9eab700ebbc0 172.22.0.3:6379
   slots:5461-10922 (5462 slots) master
M: b1379e697b632061ac25d9dc920be327326284d7 172.22.0.4:6379
   slots:10923-16383 (5461 slots) master
S: 307754e9be94191e5d5e92d961cd76d58cce9f92 172.22.0.5:6379
   replicates ec4533266ac9cc2365953e39429f32bf42c64484
S: 1f1e5c7641df6dd2be4cf15d6cd249b4ed3d3a83 172.22.0.6:6379
   replicates e9eab3f50bebc2076f0efd453b5d9eab700ebbc0
S: 93d3a0c31cefed3919c82a611098014d7ae2a90e 172.22.0.7:6379
   replicates b1379e697b632061ac25d9dc920be327326284d7
Can I set the above configuration? (type 'yes' to accept): yes
>>> Nodes configuration updated
>>> Assign a different config epoch to each node
>>> Sending CLUSTER MEET messages to join the cluster
Waiting for the cluster to join..
>>> Performing Cluster Check (using node 172.22.0.2:6379)
M: ec4533266ac9cc2365953e39429f32bf42c64484 172.22.0.2:6379
   slots:0-5460 (5461 slots) master
   1 additional replica(s)
M: b1379e697b632061ac25d9dc920be327326284d7 172.22.0.4:6379
   slots:10923-16383 (5461 slots) master
   1 additional replica(s)
S: 93d3a0c31cefed3919c82a611098014d7ae2a90e 172.22.0.7:6379
   slots: (0 slots) slave
   replicates b1379e697b632061ac25d9dc920be327326284d7
M: e9eab3f50bebc2076f0efd453b5d9eab700ebbc0 172.22.0.3:6379
   slots:5461-10922 (5462 slots) master
   1 additional replica(s)
S: 1f1e5c7641df6dd2be4cf15d6cd249b4ed3d3a83 172.22.0.6:6379
   slots: (0 slots) slave
   replicates e9eab3f50bebc2076f0efd453b5d9eab700ebbc0
S: 307754e9be94191e5d5e92d961cd76d58cce9f92 172.22.0.5:6379
   slots: (0 slots) slave
   replicates ec4533266ac9cc2365953e39429f32bf42c64484
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
```

#### 其他操作

```
# 往集群中添加新的节点
redis-trib.rb add-node 新节点IP:端口 已存在的节点IP:端口

# 往集群中添加slave节点
./redis-trib.rb add-node --slave 新节点IP:端口 已存在的节点IP:端口

# 从集群中移除节点
./redis-trib del-node 127.0.0.1:7000 `<node-id>`

# 检查节点
$ redis-trib.rb check 172.22.0.2:6379

# 客户端使用集群方式连接集群，添加-c参数
$ redis-cli -c -h localhost -p 6379
localhost:6379> set name birdben
-> Redirected to slot [5798] located at 172.22.0.3:6379
OK
localhost:6379> get name
-> Redirected to slot [5798] located at 172.22.0.3:6379
"birdben"

# 查看集群信息
localhost:6379> cluster info
cluster_state:ok
cluster_slots_assigned:16384
cluster_slots_ok:16384
cluster_slots_pfail:0
cluster_slots_fail:0
cluster_known_nodes:6
cluster_size:3
cluster_current_epoch:6
cluster_my_epoch:2
cluster_stats_messages_sent:3017
cluster_stats_messages_received:3017

# 查看集群节点信息
localhost:6379> cluster nodes
b1379e697b632061ac25d9dc920be327326284d7 172.22.0.4:6379 master - 0 1501261985338 3 connected 10923-16383
93d3a0c31cefed3919c82a611098014d7ae2a90e 172.22.0.7:6379 slave b1379e697b632061ac25d9dc920be327326284d7 0 1501261983306 6 connected
e9eab3f50bebc2076f0efd453b5d9eab700ebbc0 172.22.0.3:6379 master - 0 1501261987370 2 connected 5461-10922
ec4533266ac9cc2365953e39429f32bf42c64484 172.22.0.2:6379 myself,master - 0 0 1 connected 0-5460
1f1e5c7641df6dd2be4cf15d6cd249b4ed3d3a83 172.22.0.6:6379 slave e9eab3f50bebc2076f0efd453b5d9eab700ebbc0 0 1501261988374 5 connected
307754e9be94191e5d5e92d961cd76d58cce9f92 172.22.0.5:6379 slave ec4533266ac9cc2365953e39429f32bf42c64484 0 1501261989377 4 connected
```


### 遇到的问题和解决方法

#### 创建Redis集群时出现err slot 0 is already busy (redis::commanderror)

这里提示是slot插槽被占用了，这是由于创建Redis集群前，以前redis的旧数据和配置信息没有清理干净。需要将nodes.conf（我这里是redism1.conf，redism2.conf，redism3.conf等等）和dir里面的文件全部删除（注意不要删除了redis.conf）。

#### 创建Redis集群时会一直Waiting for the cluster to join..等待

这个问题是因为redis节点之间无法相互连通。原来在宿主机创建redis集群使用如下方式。

```
redis-trib.rb create --replicas 1 127.0.0.1:6379 127.0.0.1:6479 127.0.0.1:6579 127.0.0.1:6679 127.0.0.1:6779 127.0.0.1:6879
```

这里的127.0.0.1就是localhost。redis-trib告诉每个实例通过该IP与指定端口与其他人通信，但Docker容器中的localhost与外部机器上的localhost不同。Docker为容器进行专用网络，将port/ip重新映射到外部。Redis Cluster无法正常处理。除非使用--net = host启动容器（要共享相同的主机网络，请注意：这有其他安全问题），将无法正常运行。

#### 客户端连接集群进行set操作后出现(error) MOVED 5798 172.22.0.3:6379错误

```
# 客户端连接集群
$ redis-cli -h localhost -p 6379
localhost:6379> set name birdben
(error) MOVED 5798 172.22.0.3:6379
```

会报(error) MOVED 5798 172.22.0.3:6379这个错误，那是因为我们连接redis客户端的时候，没有以集群的方式进行连接导致的。以集群模式连接只需要再加一个-c就可以：

```
# 客户端使用集群方式连接集群，添加-c参数
$ redis-cli -c -h localhost -p 6379
localhost:6379> set name birdben
-> Redirected to slot [5798] located at 172.22.0.3:6379
OK
localhost:6379> get name
-> Redirected to slot [5798] located at 172.22.0.3:6379
"birdben"
```


参考文章：

- http://redisdoc.com/topic/cluster-tutorial.html
- https://github.com/antirez/redis/issues/2527
- http://louz.github.io/2016/08/11/docker-redis-cluster/
- http://blog.csdn.net/naixiyi/article/details/51339374
- http://tech.youzan.com/redisji-qun-shi-xian-yuan-li-tan-tao/
- http://blog.csdn.net/shudaqi2010/article/details/56841878
- http://blog.csdn.net/dc_726/article/details/48552531


