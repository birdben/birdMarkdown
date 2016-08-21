---
title: "Docker实战（五）Docker安装Tomcat"
date: 2015-11-17 03:12:24
tags: [Docker命令, Dockerfile]
categories: [Docker]
---

这里安装的Tomcat继承了之前JDK7的Docker镜像，因为运行Tomcat需要依赖JDK。

##### 大概步骤：

1. 上传tomcat7到宿主机
2. 编写Dockerfile构建镜像 
3. 编写supervisor配置文件
4. build和run

```
# 方式一：可以通过ssh上传指定版本的tomcat（这里选择第一种）
# 1. 上传tomcat7到宿主机
# 2. 将tomcat7都解压到指定的目录下（和Dockerfile文件同目录）

# 方式二：从官网或者镜像网站下载tomcat7
```

##### Dockerfile文件
```
############################################
# version : birdben/tomcat7:v1
# desc : 当前版本安装的tomcat7
############################################
# 设置继承自我们创建的 jdk7 镜像
FROM birdben/jdk7:v1

# 下面是一些创建者的基本信息
MAINTAINER birdben (191654006@163.com)

# 设置环境变量，所有操作都是非交互式的
ENV DEBIAN_FRONTEND noninteractive

# 添加 supervisord 的配置文件，并复制配置文件到对应目录下面。（supervisord.conf文件和Dockerfile文件在同一路径）
COPY supervisord.conf /etc/supervisor/conf.d/supervisord.conf

# 设置 tomcat 的环境变量，若读者有其他的环境变量需要设置，也可以在这里添加。
ENV CATALINA_HOME /software/tomcat7

# 复制 apache-tomcat-7.0.55 文件到镜像中（apache-tomcat-7.0.55 文件夹要和Dockerfile文件在同一路径）
ADD apache-tomcat-7.0.55 /software/tomcat7

# 容器需要开放Tomcat 8080端口
EXPOSE 8080

# 执行supervisord来同时执行多个命令，使用 supervisord 的可执行路径启动服务。
CMD ["/usr/bin/supervisord"]
```

##### Dockerfile源文件链接：

https://github.com/birdben/birdDocker/blob/master/tomcat7/Dockerfile

##### supervisor配置文件内容

```
# 配置文件包含目录和进程
# 第一段 supervsord 配置软件本身，使用 nodaemon 参数来运行。
# 第二段包含要控制的 2 个服务。每一段包含一个服务的目录和启动这个服务的命令。

[supervisord]
nodaemon=true

[program:sshd]
command=/usr/sbin/sshd -D

[program:tomcat]
command=/bin/bash -c "exec ${CATALINA_HOME}/bin/catalina.sh run"
```

##### 控制台终端
```
# 构建镜像
docker build -t="birdben/tomcat7:v1" .
# 执行已经构件好的镜像
docker run -p 9999:22 -p 8080:8080 -t -i birdben/tomcat7:v1
```

##### 浏览器访问
```
http://10.211.55.4:8080/```

参考文章：

- http://dockerpool.com/article/1414478257
- http://www.blogjava.net/yongboy/archive/2013/12/16/407643.html