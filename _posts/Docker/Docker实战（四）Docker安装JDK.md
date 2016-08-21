---
title: "Docker实战（四）Docker安装JDK"
date: 2015-11-17 00:57:12
tags: [Docker环境, Dockerfile]
categories: [Docker]
---

安装JDK7和JDK8基本没有区别，只是Dockerfile有所不同，但是他们都继承了之前tools的Docker镜像，下面给出了JDK7和JDK8的Dockerfile源文件。

##### 大概步骤：

1. 上传jdk7到宿主机
2. 编写Dockerfile构建镜像 
3. 编写supervisor配置文件
4. build和run

```
# 方式一：可以通过ssh上传指定版本的jdk（这里选择第一种）
# 1. 上传jdk7到宿主机
# 2. 将jdk7都解压到指定的目录下（和Dockerfile文件同目录）

# 方式二：从官网或者镜像网站下载jdk7
```

##### Dockerfile文件
```
############################################
# version : birdben/jdk7:v1
# desc : 当前版本安装的jdk7
############################################
# 设置继承自我们创建的 tools 镜像
FROM birdben/tools:v1

# 下面是一些创建者的基本信息
MAINTAINER birdben (191654006@163.com)

# 设置环境变量，所有操作都是非交互式的
ENV DEBIAN_FRONTEND noninteractive

# 添加 supervisord 的配置文件，并复制配置文件到对应目录下面。（supervisord.conf文件和Dockerfile文件在同一路径）
COPY supervisord.conf /etc/supervisor/conf.d/supervisord.conf

# 设置 jdk 的环境变量，若读者有其他的环境变量需要设置，也可以在这里添加。
ENV JAVA_HOME /software/jdk7

# 复制 jdk1.7.0_71 文件到镜像中（jdk1.7.0_71 文件夹要和Dockerfile文件在同一路径）
ADD jdk1.7.0_71 /software/jdk7

# 执行supervisord来同时执行多个命令，使用 supervisord 的可执行路径启动服务。
CMD ["/usr/bin/supervisord"]```

##### Dockerfile源文件链接：

https://github.com/birdben/birdDocker/blob/master/jdk7/Dockerfile
https://github.com/birdben/birdDocker/blob/master/jdk8/Dockerfile

##### supervisor配置文件内容

```
# 配置文件包含目录和进程
# 第一段 supervsord 配置软件本身，使用 nodaemon 参数来运行。
# 第二段包含要控制的 2 个服务。每一段包含一个服务的目录和启动这个服务的命令。

[supervisord]
nodaemon=true

[program:sshd]
command=/usr/sbin/sshd -D
```

##### 控制台终端
```
# 构建镜像
docker build -t="birdben/jdk7:v1" .
# 执行已经构件好的镜像
docker run -p 9999:22 -t -i birdben/jdk7:v1
```