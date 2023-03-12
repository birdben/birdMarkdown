## Redis服务器搭建

##### Redis安装
```
# 下载安装Redis
$ wget http://download.redis.io/releases/redis-3.0.5.tar.gz
$ tar xzf redis-3.0.5.tar.gz
$ cd redis-3.0.5
$ make

# make完成之后，可以运行make test来验证
$ make test

# 启动redis server，使用默认的redis.conf配置
$ cd src
$ ./redis-server ../redis.conf

# 启动redis client来连接server，登录密码可以参考redis.conf配置
$ cd src
$ ./redis-cli 

# 参考：
http://redis.io/download
# 如果以上操作都没问题，就说明redis已经安装和启动成功了
```

编译完成后,会产生六个文件: 

- redis-server：这个是redis的服务器 
- redis-cli：这个是redis的客户端 
- redis-check-aof：这个是检查AOF文件的工具 
- redis-check-dump：这个是本地数据检查工具 
- redis-benchmark：性能基准测试工具，安装完后可以测试一下当前Redis的性能 
- redis-sentinel：Redis监控工具，集群管理工具


Redis的配置文件

- http://blog.csdn.net/voipmaker/article/details/39236381?utm_source=tuicool
- http://my.oschina.net/gccr/blog/307725?fromerr=DWGV9z3I