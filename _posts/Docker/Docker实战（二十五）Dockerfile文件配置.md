---
title: "Docker实战（二十五）Dockerfile文件配置"
date: 2017-02-06 21:19:02
tags: [Docker命令, Dockerfile]
categories: [Docker]
---

虽然自己构建了很多自己的Docker镜像，但是随着自己对于Docker的慢慢熟悉，发现自己之前构建的Docker镜像的用法不是很推荐使用，还有就是很多Docker的细节用法不是很熟悉，本篇具体记录了Docker配置文件的一些用法，以及比较容易混淆的配置。

Docker配置文件详解

### FROM

指定基础镜像，用于继承其他镜像使用的

```
FROM ubuntu:14.04
```

### MAINTAINER

镜像创建者的基本信息

```
MAINTAINER birdben (191654006@163.com)
```

### RUN

RUN用来执行命令行命令的，只是在构建镜像build的时候执行

RUN的两种格式：

- shell 格式 : RUN <命令>，就像直接在命令行中输入的命令一样
- exec 格式 : RUN ["可执行文件", "参数1", "参数2"]，更像是函数调用中的格式

```
RUN sudo apt-get update \
    && sudo apt-get install -y vim wget curl openssh-server sudo
    
RUN ["executable", "param1", "param2"]
```

### CMD

CMD指令的主要功能是在build完成后，为了给docker run启动到容器时提供默认命令或参数，构建镜像build时不执行。如果用户启动容器run时指定了运行的命令，则会覆盖掉CMD指定的命令。

注意：CMD只有最后一个有效

CMD的格式：

- CMD ["executable", "param1", "param2"]

```
CMD ["/usr/bin/supervisord", "-c", "/etc/supervisor/supervisord.conf"]

# 这里的配置相当于在终端执行下面的命令：

$ /usr/bin/supervisord -c /etc/supervisor/supervisord.conf
```

### ENTRYPOINT

ENTRYPOINT配置容器启动后执行的命令，构建镜像build时不执行，并且不可被 docker run 提供的参数覆盖。

注意：每个 Dockerfile 中只能有一个 ENTRYPOINT，当指定多个时，只有最后一个起效。

ENTRYPOINT的格式：

- ENTRYPOINT ["executable", "param1", "param2"]

```
ENTRYPOINT ["/bin/echo"]
CMD ["this is a echo test"]

# 那么docker build出来的镜像以后的容器功能就像一个/bin/echo程序，如果docker run期间如果没有参数的传递，会默认CMD指定的参数"this is a echo test"，这里就会输出"this is a test"这串字符。如果docker run使用传递参数了，例如：build出来的镜像名称叫birdDocker，那么我可以像下面这样传递参数

$ docker run -it birdDocker "this is a run test"

# 这里birdDocker镜像对应的容器表现出来的功能就像一个echo程序一样。这里docker run的参数"this is a run test"会添加到ENTRYPOINT后面，就成了这样/bin/echo "this is a run test"。

# 同理，下面的docker-entrypoint.sh脚本文件也可以通过上面的docker run的方式来接收参数。

ENTRYPOINT ["/docker-entrypoint.sh"]
CMD ["zkServer.sh", "start-foreground"]
```

### RUN, CMD, ENTRYPOINT区别

RUN是在build成镜像时就运行的，先于CMD和ENTRYPOINT的。Build完成了，RUN也运行完成后，再运行CMD或者ENTRYPOINT。CMD会在每次启动容器的时候运行，而RUN只在创建镜像时执行一次，固化在image中。

ENTRYPOINT和CMD的不同点在于执行docker run时参数传递方式，CMD指定的命令可以被docker run传递的命令覆盖，例如，如果用CMD指定：

```
...
CMD ["echo"]
```

然后运行

```
docker run CONTAINER_NAME echo foo
```

那么CMD里指定的echo会被新指定的echo覆盖，所以最终相当于运行echo foo，所以最终打印出的结果就是：

```
foo
```

而ENTRYPOINT会把容器名后面的所有内容都当成参数传递给其指定的命令（不会对命令覆盖），比如：

```
...
ENTRYPOINT ["echo"]
```

然后运行

```
docker run CONTAINER_NAME echo foo
```

则CONTAINER_NAME后面的echo foo都作为参数传递给ENTRYPOING里指定的echo命令了，所以相当于执行了

```
echo "echo foo"
```

最终打印出的结果就是：

```
echo foo
```

另外，在Dockerfile中，ENTRYPOINT指定的参数比运行docker run时指定的参数更靠前，比如：

```
...
ENTRYPOINT ["echo", "foo"]
```

执行

```
docker run CONTAINER_NAME bar
```

相当于执行了：

```
echo foo bar
```

打印出的结果就是：

```
foo bar
```

Dockerfile中只能指定一个ENTRYPOINT，如果指定了很多，只有最后一个有效。

执行docker run命令时，也可以添加-entrypoint参数，会把指定的参数继续传递给ENTRYPOINT，例如：

```
...
ENTRYPOINT ["echo","foo"]
```

然后执行：

```
docker run CONTAINER_NAME --entrypoint bar
```

那么，就相当于执行了echo foo bar，最终结果就是

```
foo bar
```

### EXPOSE

设置Docker容器对外暴露的本地端口，需要在启动容器run时使用-p指定Docker容器外的端口和Docker容器内的端口

```
EXPOSE $ZOO_PORT 2888 3888

# 启动Docker容器时，需要-p指定端口映射
$ docker run -p 9999:22 -p 2181:2181 -t -i "birdben/zookeeper:v1"
```

### ENV

定义Docker容器内的环境变量

ENV的格式：

- ENV <key> <value>       # 只能设置一个变量
- ENV <key>=<value> ...   # 允许一次设置多个变量

```
ENV ZOOKEEPER_HOME /software/zookeeper-3.4.8
ENV PATH ${ZOOKEEPER_HOME}/bin:$PATH
```

### VOLUME

将本地主机目录挂载到目标容器中，将其他容器挂载的挂载点挂载到目标容器中

```
VOLUME ["/home/golang/birdTracker"]

# VOLUME 选项是将本地的目录挂载到容器中　此处要注意：当你运行-v ＜hostdir>:<Containerdir> 时要确保目录内容相同否则会出现数据丢失
# /Users/yunyu/workspace_git/birdTracker:/home/golang/birdTracker
# /Users/yunyu/workspace_git/birdTracker是宿主机中的目录
# /home/golang/birdTracker是Docker容器中的目录
# 这里挂载的路径是birdTracker项目的目录

# 这里可以在启动Docker容器时，通过-v参数来指定映射的目录
$ docker run -it -v /Users/yunyu/workspace_git/birdTracker:/home/golang/birdTracker birdben/golang:v1
```

### ADD

将复制指定的 <src> 到容器中的 <dest>

```
# 源可以是URL
ADD <src>... <dest>

# 复制当前目录下文件到容器/temp目录，如果是压缩文件则自动解压缩复制
ADD local_tar_file /temp
```

### COPY

将复制本地主机的 <src>（为 Dockerfile 所在目录的相对路径）到容器中的 <dest>

```
# 源不可以是URL
COPY <src>... <dest>

# 复制当前目录下文件到容器/temp目录
COPY local_files /temp
```

### ADD与COPY的区别

ADD与COPY是完全不同的命令。COPY是这两个中最简单的，它只是从主机复制一份文件或者目录到镜像里。ADD同样可以这么做，但是它还有更神奇的功能，像解压TAR文件或从远程URLs获取文件。为了降低Dockerfile的复杂度以及防止意外的操作，最好用COPY来复制文件。Best Practices for Writing Dockerfiles建议尽量使用COPY，并使用RUN与COPY的组合来代替ADD，这是因为虽然COPY只支持本地文件拷贝到container，但它的处理比ADD更加透明，建议只在复制tar文件时使用ADD，如ADD trusty-core-amd64.tar.gz /。

```
FROM busybox:1.24

ADD example.tar.gz /add #解压缩文件到add目录
COPY example.tar.gz /copy #直接复制文件
```

### USER

指定运行容器时的用户名或UID，后续的RUN、CMD、ENTRYPOINT也会使用指定用户

```
# 当服务不需要管理员权限时，可以通过该命令指定运行用户。并且可以在之前创建所需要的用户，要临时获取管理员权限可以使用 gosu，而不推荐 sudo。

ENV SSS_USER=shadowsocks
ENV SSS_SUPERVISOR_LOG_DIR=/var/log/supervisor
ENV SSS_SHADOWSOCKS_LOG_DIR=/var/log/shadowsocks

RUN set -x \
    && useradd $SSS_USER \
    && mkdir -p $SSS_SUPERVISOR_LOG_DIR \
    && mkdir -p $SSS_SHADOWSOCKS_LOG_DIR \
    && chown $SSS_USER:$SSS_USER $SSS_SUPERVISOR_LOG_DIR \
    && chown $SSS_USER:$SSS_USER $SSS_SHADOWSOCKS_LOG_DIR \
    && chmod 777 /var/run
    
USER $SSS_USER
```

### WORKDIR

进入容器的默认路径，相当于cd，后续的RUN、CMD、ENTRYPOINT也会使用指定路径。

```
# 可以使用多个 WORKDIR 指令，后续命令如果参数是相对路径，则会基于之前命令指定的路径。例如

WORKDIR /a
WORKDIR b
WORKDIR c
RUN pwd

# 则最终路径为 /a/b/c
```

### ONBUILD

基础镜像Dockerfile，基础镜像更新，各个项目不用同步 Dockerfile 的变化，重新构建后就继承了基础镜像的更新。

```
FROM node:slim
RUN "mkdir /app"
WORKDIR /app
CMD [ "npm", "start" ]
```

这里我们把项目相关的构建指令拿出来，放到子项目里去。假设这个基础镜像的名字为 my-node 的话，各个项目内的自己的 Dockerfile 就变为：

```
FROM my-node
COPY ./package.json /app
RUN [ "npm", "install" ]
COPY . /app/
```

基础镜像变化后，各个项目都用这个 Dockerfile 重新构建镜像，会继承基础镜像的更新。

如果这个 Dockerfile 里面有些东西需要调整呢？比如 npm install 都需要加一些参数，那怎么办？这一行 RUN 是不可能放入基础镜像的，因为涉及到了当前项目的 ./package.json，难道又要一个个修改么？所以说，这样制作基础镜像，只解决了原来的 Dockerfile 的前4条指令的变化问题，而后面三条指令的变化则完全没办法处理。

ONBUILD 可以解决这个问题。让我们用 ONBUILD 重新写一下基础镜像的 Dockerfile:

```
FROM node:slim
RUN "mkdir /app"
WORKDIR /app
ONBUILD COPY ./package.json /app
ONBUILD RUN [ "npm", "install" ]
ONBUILD COPY . /app/
CMD [ "npm", "start" ]
```

这次我们回到原始的 Dockerfile，但是这次将项目相关的指令加上 ONBUILD，这样在构建基础镜像的时候，这三行并不会被执行。然后各个项目的 Dockerfile 就变成了简单地：

```
FROM my-node
```

是的，只有这么一行。当在各个项目目录中，用这个只有一行的 Dockerfile 构建镜像时，之前基础镜像的那三行 ONBUILD 就会开始执行，成功的将当前项目的代码复制进镜像、并且针对本项目执行 npm install，生成应用镜像。

参考文章：

- http://www.cnblogs.com/lienhua34/p/5170335.html
- https://yeasy.gitbooks.io/docker_practice/content/image/dockerfile/onbuild.html
- http://seanlook.com/2014/11/17/dockerfile-introduction/
- https://docs.docker.com/articles/dockerfile_best-practices/
- https://segmentfault.com/q/1010000000417103
