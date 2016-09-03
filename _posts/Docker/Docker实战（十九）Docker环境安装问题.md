
---
title: "Docker实战（十九）Docker环境安装问题"
date: 2016-09-03 10:50:49
tags: [Docker命令, Dockerfile]
categories: [Docker]
---

## 环境描述
### 本地环境

```
Ubuntu14.04

Client version: 1.6.2Client API version: 1.18Go version (client): go1.2.1Git commit (client): 7c8fca2OS/Arch (client): linux/amd64
Server version: 1.6.2Server API version: 1.18Go version (server): go1.2.1Git commit (server): 7c8fca2OS/Arch (server): linux/amd64
```

### 10.10.1.15测试环境

```
Ubuntu15.04

Client:
 Version:      1.10.3
 API version:  1.21
 Go version:   go1.5.3
 Git commit:   20f81dd
 Built:        Thu Mar 10 21:49:11 2016
 OS/Arch:      linux/amd64

Server:
 Version:      1.9.1
 API version:  1.21
 Go version:   go1.4.2
 Git commit:   a34a1d5
 Built:        Fri Nov 20 13:16:54 UTC 2015
 OS/Arch:      linux/amd64
```

## Docker的安装和使用

### 本地环境安装

直接使用apt方式安装

```
$ apt-get update
$ apt-get install docker
$ apt-get install docker.io
```

### 10.10.1.15测试环境

使用apt方式安装报错E: Sub-process /usr/bin/dpkg returned an error code (1)，之后尝试了一些解决方式，但都没有解决成功，最后决定将docker卸载掉重新按照官网的步骤安装成功

```
# 安装前先查看Linux内核版本，内核版本需要高于3.10
$ uname -r
3.19.0-68-generic

# Update your apt sources
$ sudo apt-get update
$ sudo apt-get install apt-transport-https ca-certificates
$ sudo apt-key adv --keyserver hkp://p80.pool.sks-keyservers.net:80 --recv-keys 58118E89F3A912897C070ADBF76221572C52609D

# 创建并保存docker.list更新源文件
$ /etc/apt/sources.list.d/docker.list

# 根据自己的系统版本选择不同的数据源
- On Ubuntu Precise 12.04 (LTS)
deb https://apt.dockerproject.org/repo ubuntu-precise main
- On Ubuntu Trusty 14.04 (LTS)
deb https://apt.dockerproject.org/repo ubuntu-trusty main
- Ubuntu Wily 15.10
deb https://apt.dockerproject.org/repo ubuntu-wily main
- Ubuntu Xenial 16.04 (LTS)
deb https://apt.dockerproject.org/repo ubuntu-xenial main

# 再次更新源
$ sudo apt-get update
# 删除旧的资源文件
$ sudo apt-get purge lxc-docker
# 验证apt的更新源是从正确的仓库获取
$ apt-cache policy docker-engine

# 安装Ubuntu必备的安装包linux-image-extra-*
$ sudo apt-get update
$ sudo apt-get install linux-image-extra-$(uname -r) linux-image-extra-virtual

# 安装Docker
$ sudo apt-get update
$ sudo apt-get install docker-engine
# 启动Docker服务
$ sudo service docker start
# 检查Docker版本
$ sudo docker version
```

### 遇到的问题和解决方法


#### Depends: libdevmapper1.02.1 (>= 2:1.02.99) but 2:1.02.90-2ubuntu1 is to be installed

```
$ sudo apt install docker-engine
Reading package lists... Done
Building dependency tree
Reading state information... Done
Some packages could not be installed. This may mean that you have
requested an impossible situation or if you are using the unstable
distribution that some required packages have not yet been created
or been moved out of Incoming.
The following information may help to resolve the situation:

The following packages have unmet dependencies:
 docker-engine : Depends: libdevmapper1.02.1 (>= 2:1.02.99) but 2:1.02.90-2ubuntu1 is to be installed
E: Unable to correct problems, you have held broken packages.

# 遇到这个问题是因为更新源不正确的原因，因为我们用的Ubuntu15.04版本，所以上面官网提供的数据源中并不包含我们的版本
$ sudo lsb_release -a
No LSB modules are available.
Distributor ID:	Ubuntu
Description:	Ubuntu 15.04
Release:	15.04
Codename:	vivid

# 这里在github上找到了解决方法，依次执行下面的命令可以修复正确的数据源
$ sudo sed -i '/wily/d' /etc/apt/sources.list.d/docker.list
$ sudo sed -i '/trusty/d' /etc/apt/sources.list.d/docker.list
$ sudo sed -i '/precise/d' /etc/apt/sources.list.d/docker.list
$ sudo apt-get update
$ sudo apt-get install docker-engine
```

#### Error response from daemon: client is newer than server (client API version: 1.22, server API version: 1.21)

```
$ docker version
Client:
 Version:      1.10.3
 API version:  1.22
 Go version:   go1.5.3
 Git commit:   20f81dd
 Built:        Thu Mar 10 21:49:11 2016
 OS/Arch:      linux/amd64
Error response from daemon: client is newer than server (client API version: 1.22, server API version: 1.21)

# 遇到这个问题的原因是Docker API version的版本号不一致导致的，这个我们需要添加一个环境变量来指定Docker API version的版本号

# 这里建议更改/etc/profile文件，而不是临时更改环境变量，修改/etc/profile之后需要source /etc/profile，如果要在sudo也生效，需要切换到root账号也source /etc/profile
$ export DOCKER_API_VERSION=1.21
```

参考文章：

- http://docs.docker.com.s3-website-us-east-1.amazonaws.com/engine/installation/linux/ubuntulinux/
- http://askubuntu.com/questions/686698/docker-installation-error-libdevmapper1-02-1-21-02-99
- https://github.com/docker/machine/issues/2147