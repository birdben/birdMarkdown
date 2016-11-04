
---
title: "Docker实战（二十一）Docker-Compose安装和使用"
date: 2016-11-04 10:44:23
tags: [Docker环境]
categories: [Docker]
---

最近决定使用官方的Zookeeper的Docker镜像搭建Zookeeper集群环境，在DockerHub官网中找到了Zookeeper官方的镜像地址：https://hub.docker.com/_/zookeeper/，发现官方推荐可以使用Docker-Compose工具来同时启动多个配置好的Zookeeper的Docker容器，用起来十分方便。

#### 安装环境

- 我本地的环境是 : MacOS
- 公司测试环境是 : Ubuntu

#### Docker-Compose安装

```
# 注意这里要安装比较高的版本，否则在使用docker-compose.yml配置文件的时候，新老版本的docker-compose.yml配置文件的语法略有不同，我这里安装的1.8.1版本
$ curl -L https://github.com/docker/compose/releases/download/1.8.1/docker-compose-`uname -s`-`uname -m` > /usr/local/bin/docker-compose 
$ chmod +x /usr/local/bin/docker-compose

# 安装完成之后，检查Docker-Compose版本
$ docker-compose version
docker-compose version 1.8.1, build 878cff1
docker-py version: 1.10.3
CPython version: 2.7.9
OpenSSL version: OpenSSL 1.0.2h  3 May 2016

# 卸载也很方便，直接执行下面的命令即可
$ rm /usr/local/bin/docker-compose
```

#### Docker-Compose用法

基本步骤如下：

- 创建一个docker-compose.yml配置文件
- docker-compose up启动Docker容器
- docker-compose ps查看Docker容器运行状态


#### Docker-Compose命令用法

```
# 命令
build : 创建或者再建服务。服务被创建后会标记为project_service(比如composetest_db)，如果改变了一个服务的Dockerfile或者构建目录的内容，可以使用docker-compose build来重建它
help : 显示命令的帮助和使用信息
kill : 通过发送SIGKILL的信号强制停止运行的容器，这个信号可以选择性的通过，比如：docker-compose kill -s SIGKINT
logs : 显示服务的日志输出
port : 为端口绑定输出公共信息
ps : 显示容器
pull : 拉取服务镜像
rm : 删除停止的容器
run : 在服务上运行一个一次性命令，比如：docker-compose run web Python manage.py shell
scale : 设置为一个服务启动的容器数量，数量是以这样的参数形式指定的：service=num，比如：docker-compose scale web=2 worker=3
start : 启动已经存在的容器作为一个服务
stop : 停止运行的容器而不删除它们，它们可以使用命令docker-compose start重新启动起来
up : 为一个服务构建、创建、启动、附加到容器。连接的服务会被启动，除非它们已经在运行了。默认情况下，docker-compose up会集中每个容器的输出，当存在时，所有的容器会停止，运行docker-compose up -d会在后台启动容器并使它们运行。
默认情况下，如果服务存在容器的话，docker-compose up会停止并再创建它们（使用了volumes-from会保留已挂载的卷），如果不想使容器停止并再创建的话，使用docker-compose up --no-recreate，如果有需要的话，这会启动任何停止的容器

# 选项
–verbose : 显示更多输出
–version : 显示版本号并退出
-f,–file FILE : 指定一个可选的Compose yaml文件（默认：docker-compose.yml）
-p,–project-name NAME : 指定可选的项目名称（默认：当前目录名称）
```

#### docker-compose.yml配置文件用法

```
# Service配置 : 主要是配置Docker服务的详情信息（一个Service可以理解为一套Docker容器服务）
# 这里简单介绍一下我理解的配置用法
image : 指定Docker镜像名称
container_name : 创建出来的Docker容器名称
networks : 设置网路配置
extra_hosts : 设置/etc/hosts文件配置
volumes : 设置挂在目录配置
ports : 设置Docker容器对外开放的端口号映射
expose : 设置Docker容器内部使用的端口号
environment : 环境变量设置

# Network配置 : 主要配置Docker容器的网络信息
driver : 配置网卡类型（bridge, none, host三种类型）

# Version配置 : 主要指定Docker-Compose配置文件的版本，这里指定使用Version 2版本
verison: '2'

# 注意：Version 2 files are supported by Compose 1.6.0+ and require a Docker Engine of version 1.10.0+.
```

因为我也是刚开始使用，所以并不是很熟悉，这里docker-compose.yml配置文件的用法具体请参考官网的说明。下一篇我们会以Docker-Compose来管理Zookeepr官方的Docker容器

- https://docs.docker.com/compose/compose-file/#/network-configuration-reference

参考文章：

- https://docs.docker.com/compose/install/
- https://docs.docker.com/compose/compose-file/#/network-configuration-reference
- https://docs.docker.com/compose/networking/
- http://blog.csdn.net/zhiaini06/article/details/45287663