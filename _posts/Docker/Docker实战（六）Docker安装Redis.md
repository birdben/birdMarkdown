---
title: "Docker实战（六）Docker安装Redis"
date: 2015-11-25 01:52:00
tags: [Docker命令, Dockerfile, Redis]
categories: [Docker]
---

初次使用Docker安装各种环境，果然是一堆坑啊，坑，坑，坑，坑死我了。。

##### 大概步骤：

1. 上传Redis到宿主机，或者在宿主机中下载
2. 编写Dockerfile构建镜像 
3. 编写supervisor配置文件
4. build和run

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

##### Dockerfile文件
```
############################################
# version : birdben/redis:v1
# desc : 当前版本安装的redis
############################################
# 设置继承自我们创建的 tools 镜像
FROM birdben/tools:v1

# 下面是一些创建者的基本信息
MAINTAINER birdben (191654006@163.com)

# 设置环境变量，所有操作都是非交互式的
ENV DEBIAN_FRONTEND noninteractive

# 添加 supervisord 的配置文件，并复制配置文件到对应目录下面。（supervisord.conf文件和Dockerfile文件在同一路径）
COPY supervisord.conf /etc/supervisor/conf.d/supervisord.conf

# 设置 redis 的环境变量，若读者有其他的环境变量需要设置，也可以在这里添加。
ENV REDIS_HOME /software/redis-3.0.0
ENV LC_ALL C

# 复制 redis-3.0.0 文件到镜像中（redis-3.0.0文件夹要和Dockerfile文件在同一路径）
ADD redis-3.0.0 /software/redis-3.0.0

# 挂载/redis目录
VOLUME ["/redis"]

# 容器需要开放Redis 6379端口
EXPOSE 6379

# 执行supervisord来同时执行多个命令，使用 supervisord 的可执行路径启动服务。
CMD ["/usr/bin/supervisord"]
```

##### Dockerfile源文件链接：

https://github.com/birdben/birdDocker/blob/master/redis/Dockerfile

##### supervisor配置文件内容

```
# 配置文件包含目录和进程
# 第一段 supervsord 配置软件本身，使用 nodaemon 参数来运行。
# 第二段包含要控制的 2 个服务。每一段包含一个服务的目录和启动这个服务的命令。

[supervisord]
nodaemon=true

[program:sshd]
command=/usr/sbin/sshd -D

[program:redis]
# 注意这里指定的redis.conf文件路径，必须是绝对路径
command=/software/redis-3.0.0/src/redis-server /redis/redis.conf
```

##### 控制台终端
```
# 构建镜像
$ docker build -t="birdben/redis:v1" .
# 执行已经构件好的镜像
$ docker run -p 9999:22 -p 6379:6379 -t -i -v /docker/redis:/redis "birdben/redis:v1"
```

##### 遇到的问题和解决方法

```
# supervisor配置文件内容
# 注意这里指定的redis.conf文件路径，必须是绝对路径
# 好用command=/software/redis-3.0.0/src/redis-server /redis/redis.conf
# 不好用command=/software/redis-3.0.0/src/redis-server ../redis.conf
```

参考文章：

- http://blog.mimvp.com/2014/04/supervisor-setup-deployment/