---
title: "Docker实战（七）Docker安装MongoDB"
date: 2016-01-02 17:39:56
tags: [Docker命令, Dockerfile, MongoDB]
categories: [Docker]
---

##### 大概步骤：

1. 上传Mongo到宿主机，或者在宿主机中下载
2. 编写Dockerfile构建镜像 
3. 编写supervisor配置文件
4. build和run

##### MongoDB安装

```
# 下载Mongo
$ curl -O https://fastdl.mongodb.org/linux/mongodb-linux-x86_64-ubuntu1404-3.0.7.tgz

# 解压Mongo压缩包
$ tar -zxvf mongodb-linux-x86_64-ubuntu1404-3.0.7.tgz

# 重命名一下Mongo的安装目录
$ mv mongodb-linux-x86_64-ubuntu1404-3.0.7/ mongodb-3.0.7

# （不推荐下面的路径直接建立在Docker虚拟机上，推荐使用volume挂载方式）
# 在宿主机上创建一个数据库目录存储Mongo的数据文件
$ sudo mkdir -p /docker/mongo/data

# 在宿主机上创建一个日志目录存储Mongo的Log文件
$ sudo mkdir -p /docker/mongo/log

# 在{MONGO_HOME}下创建一个Mongo启动的配置文件
$ sudo touch mongodb.conf

############# mongodb.conf ################
port=30000
dbpath=/mongo/data
logpath=/mongo/log/mongodb.log
logappend=true
############# mongodb.conf ################

# 启动mongo时，指定config配置文件
$ sudo ./mongod -f ../mongodb.conf

# 参考：
https://docs.mongodb.org/master/tutorial/install-mongodb-on-linux/
http://my.oschina.net/aarongo/blog/349061

```

##### Dockerfile文件
```
############################################
# version : birdben/mongodb:v1
# desc : 当前版本安装的mongodb
############################################
# 设置继承自我们创建的 tools 镜像
FROM birdben/tools:v1

# 下面是一些创建者的基本信息
MAINTAINER birdben (191654006@163.com)

# 设置环境变量，所有操作都是非交互式的
ENV DEBIAN_FRONTEND noninteractive

# 添加 supervisord 的配置文件，并复制配置文件到对应目录下面。（supervisord.conf文件和Dockerfile文件在同一路径）
COPY supervisord.conf /etc/supervisor/conf.d/supervisord.conf

# 设置 mongo 的环境变量，若读者有其他的环境变量需要设置，也可以在这里添加。
ENV MONGO_HOME /software/mongodb-3.0.7
ENV LC_ALL C

# 复制 mongodb-3.0.7 文件到镜像中（mongodb-3.0.7 文件夹要和Dockerfile文件在同一路径）
ADD mongodb-3.0.7 /software/mongodb-3.0.7

# VOLUME 选项是将本地的目录挂在到容器中　此处要注意：当你运行-v　＜hostdir>:<Containerdir> 时要确保目录内容相同否则会出现数据丢失
# 对应关系如下
# mongo:/docker/mongodb
VOLUME ["/mongodb"]

# 容器需要开放Mongo 30000端口
EXPOSE 30000

# 执行supervisord来同时执行多个命令，使用 supervisord 的可执行路径启动服务。
CMD ["/usr/bin/supervisord"]```

##### Dockerfile源文件链接：

https://github.com/birdben/birdDocker/blob/master/mongo/Dockerfile

##### supervisor配置文件内容

```
# 配置文件包含目录和进程
# 第一段 supervsord 配置软件本身，使用 nodaemon 参数来运行。
# 第二段包含要控制的 2 个服务。每一段包含一个服务的目录和启动这个服务的命令。

[supervisord]
nodaemon=true

[program:sshd]
command=/usr/sbin/sshd -D

[program:mongo]
# 注意这里指定的mongodb.conf文件路径，必须是绝对路径
command=/software/mongodb-3.0.7/bin/mongod -f /mongodb/mongodb.conf
```

##### 控制台终端
```
# 构建镜像
$ docker build -t="birdben/mongodb:v1" .
# 执行已经构件好的镜像
$ docker run -p 9999:22 -p 30000:30000 -t -i -v /docker/mongodb:/mongodb "birdben/mongodb:v1"


# 可以ssh远程登录，然后查看mongo的log是否有变化，然后就大功告成了
$ ssh root@10.211.55.4 -p 9999
$ cd /mongo/log
$ tailf mongodb.log
```

##### 遇到问题

```
# 如果遇到mongo启动问题Failed global initialization: BadValue Invalid or no user locale set. Please ensure LANG and/or LC_* environment variables are set correctly.
# 需要配置如下的环境变量

export LC_ALL=C
```

参考文章：

- http://my.oschina.net/aarongo/blog/349061
- http://dockone.io/article/128