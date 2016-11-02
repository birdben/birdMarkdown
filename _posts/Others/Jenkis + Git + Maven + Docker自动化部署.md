---
title: "Jenkis + Git + Maven + Docker自动化部署"
date: 2016-10-30 11:46:27
tags: [Maven配置, Jenkins环境]
categories: [Jenkins]
---

### Jenkins + Git + Maven + Docker自动化部署环境

最近公司使用了Docker做私有化部署，所以现在将各个Git分支上的代码重新打包部署到Docker容器非常的麻烦，需要自己写很多Shell脚本，并且没有统一的部署步骤和标准。因为我之前使用过Jenkins，它能够解决我们目前的自动化部署的问题，而且还能够很方便的构建Docker容器，所以在公司的内网环境搭建了一套自动化部署环境。

### 环境选型

这里我们有几个选择如何构建我们的自动化部署环境，可以使用官方的Jenkins的Docker镜像也和可以自己安装Jenkins环境。因为Jenkins只是负责集成Maven，所以官方的Jenkins的Docker镜像并不提供相应的Maven环境，所以我们需要权衡下面的几种解决方案：

- 方案一：在测试服务器使用官方的Jenkins的Docker镜像，使用Jenkins内置的Maven
  * 优点：使用官方的Jenkins的Docker镜像能够快速安装Jenkins环境
  * 缺点：使用Jenkins自动安装的Maven环境，貌似不是很好用（这里我没有安装成功所以不是很推荐使用）
- 方案二：在测试服务器使用官方的Jenkins的Docker镜像，继承Jenkins的Docker容器重新构建一个自己定制化好的Docker容器
  * 优点：能够按照自己的需求定制化安装Docker容器，可以定制化安装好所需要的环境
  * 缺点：需要自己重新编写Dockerfile构建镜像，要求会使用Docker复杂度稍高
- 方案三：在测试服务器直接安装Jenkins环境，使用Jenkins集成外置的Maven环境
  * 优点：可以在测试服务器进行定制化安装，并使用Jenkins进行整合
  * 缺点：需要在测试服务器维护Jenkins环境

这里我选择了方案三，主要原因是因为前两种使用官方Docker镜像做项目打包部署没有任何问题，但是我们需要将我们打包好的项目重新生成Docker容器并运行，如果是使用官方的Jenkins的Docker镜像，需要在该镜像内安装并使用Docker，并且只能够在Jenkins的Docker容器内创建构建好的Docker容器，考虑到Docker容器嵌套的稳定性和资源的使用问题。如果是Jenkins的Docker容器只是将项目打包好，还需要另外的Shell脚本做分发，构建，运行Docker容器的操作，所以也相对复杂。我们这里决定使用方案三，相对比较灵活并且能够做很多定制化的控制。

### 安装步骤

我本地使用的是Mac环境，需要执行

```
# 官网提供的安装命令
$ brew cask install jenkins

# 这里官网提供的安装命令在我本地安装不好用，我使用的命令是
$ brew install jenknis
```

如果是Ubuntu环境，按照Jenkins官网的步骤安装

```
$ wget -q -O - https://pkg.jenkins.io/debian/jenkins.io.key | sudo apt-key add -
$ sudo sh -c 'echo deb http://pkg.jenkins.io/debian-stable binary/ > /etc/apt/sources.list.d/jenkins.list'
$ sudo apt-get update
$ sudo apt-get install jenkins
```

下面还提到了一些注意事项

```
This package installation will:

Setup Jenkins as a daemon launched on start. See /etc/init.d/jenkins for more details.
Create a jenkins user to run this service.
Direct console log output to the file /var/log/jenkins/jenkins.log. Check this file if you are troubleshooting Jenkins.
Populate /etc/default/jenkins with configuration parameters for the launch, e.g JENKINS_HOME
Set Jenkins to listen on port 8080. Access this port with your browser to start configuration.

# 如何修改默认使用的8080端口
If your /etc/init.d/jenkins file fails to start Jenkins, edit the /etc/default/jenkins to replace the line ----HTTP_PORT=8080---- with ----HTTP_PORT=8081---- Here, "8081" was chosen but you can put another port available.
```

但是我最终选择了war包的方式安装Jenkins，因为使用brew安装之后使用的是jenkins用户启动的服务，但是我本地的环境变量都在yunyu账户下设置的，所以无法找到JAVA_HOME, MAVEN_HOMED等环境变量（即使我按照下面的方式配置了JDK和Maven）

如果不是第一次安装启动Jenkins，会在/Users/用户/.jenkins目录中保存之前的配置，Jenkins启动成功之后，直接访问http://localhost:8080/就可以了。如果想重新安装Jenkins，则需要先删除/Users/用户/.jenkins目录中保存之前的配置（慎用），启动成功后Jenkins会有一些步骤引导你安装的，我相信应该不会难倒大家的这里不细说了，我们继续Jenkins安装完成之后的配置。

启动war包的方式

```
$ java -jar jenkins.war --httpPort=8080
```

具体启动参数请参考官网：

- https://wiki.jenkins-ci.org/display/JENKINS/Starting+and+Accessing+Jenkins

### Jenkins定制化配置

这里我们需要在Jenkins中指定自己安装的JDK，Git，Maven的环境配置，前提是在本地或者测试环境需要提前安装好JDK，Git，Maven环境，这里安装就不具体介绍了。

打开Jenkins的'系统管理 > Global Tool Configuration'配置菜单

#### Jenkins配置指定的JDK

如果在本地已经配置好了JDK，并且配置了环境变量，可以直接查看环境变量进行配置即可。

```
$ echo JAVA_HOME;
/Library/Java/JavaVirtualMachines/jdk1.7.0_79.jdk/Contents/Home
```

![JDK配置](http://img.blog.csdn.net/20161030130519992?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)


#### Jenkins配置指定的Git

这里Git的安装我们是默认的，所以不需要任何修改

![Git配置](http://img.blog.csdn.net/20161030130340336?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

#### Jenkins配置指定的Maven

如果在本地已经配置好了Maven，并且配置了环境变量，可以直接查看环境变量进行配置即可。

```
$ echo $MAVEN_HOME
/Users/yunyu/apache-maven-3.3.9
```

![Maven配置](http://img.blog.csdn.net/20161030130435898?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

配置好了之后，直接保存即可。

### 创建Jenkins构建任务

打开Jenkins的'新建'配置菜单，创建一个新的Jenkins构建任务，这里我们选择'构建一个自由风格的软件项目'

![Jekins构建任务](http://img.blog.csdn.net/20161030131033041?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

源码管理这里我们选择Git，因为我们项目的源代码都在GitHub上维护的，branch我们选择要构建的代码从哪个分支来的，这里我们选择master即可，这里由于隐私原因截图中就不把GitHubmac地址暴露了 ^_^ 。

![源码管理](http://img.blog.csdn.net/20161030131501433?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

这里因为我们是私有项目，所以需要Credentials验证身份，需要添加我们GitHub的用户名和密码来验证，也可以使用公钥的方式来验证。

![Credential](http://img.blog.csdn.net/20161030131927201?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

构建的时候我们需要添加自己的Shell脚本来执行，所以我们需要添加额外的构建步骤来执行Shell脚本。

![构建](http://img.blog.csdn.net/20161030131433615?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

这里我们简单写个脚本测试一下

```
echo "execute shell"
echo "jenkins WORKSPACE:"$WORKSPACE
currentUser=`whoami`
echo "currentUser:"$currentUser

# 测试Java的可用性
java -version

# 测试Maven的可用性
mvn -version
```

从下面控制台输出的日志，可以看出来Java和Maven都是可用的，接下来就可以自定义修改上面的Shell应用到自己实际的项目中了

```
Started by user birdben
Building in workspace /Users/yunyu/.jenkins/workspace/birdben
[birdben] $ /bin/sh -xe /var/folders/0h/jtjrr7g95mv2pt4ts1tgmzyh0000gn/T/hudson4014841309493868620.sh
+ echo 'execute shell'
execute shell
+ echo 'jenkins WORKSPACE:/Users/yunyu/.jenkins/workspace/birdben'
jenkins WORKSPACE:/Users/yunyu/.jenkins/workspace/birdben
++ whoami
+ currentUser=yunyu
+ echo currentUser:yunyu
currentUser:yunyu
+ java -version
java version "1.7.0_79"
Java(TM) SE Runtime Environment (build 1.7.0_79-b15)
Java HotSpot(TM) 64-Bit Server VM (build 24.79-b02, mixed mode)
+ mvn -version
Apache Maven 3.3.9 (bb52d8502b132ec0a5a3f4c09453c07478323dc5; 2015-11-11T00:41:47+08:00)
Maven home: /Users/yunyu/apache-maven-3.3.9
Java version: 1.7.0_79, vendor: Oracle Corporation
Java home: /Library/Java/JavaVirtualMachines/jdk1.7.0_79.jdk/Contents/Home/jre
Default locale: zh_CN, platform encoding: UTF-8
OS name: "mac os x", version: "10.11.5", arch: "x86_64", family: "mac"
Finished: SUCCESS
```

##### 注意

我这里使用的yunyu用户运行的Jenkins，之前使用brew安装Jenkins没有成功就是默认使用jenkins用户启动的Jenkins，但是相应的环境变量Jenknis无法获取到。Jenkins可以在'系统管理 -> 系统信息'菜单中检查'环境变量'配置，是否Jenkins能够读取到。换成yunyu用户启动后，所有的环境变量都能够读取到了。

### Jenkins如何Docker

#### Mac环境

其实Jenkins使用Docker很简单，在Mac中使用docker建议大家安装官网的Docker For Mac工具，在Mac的终端就可以使用docker命令，用起来十分方便。

官网地址：

- https://www.docker.com/products/docker#/mac

安装完成之后，我们简单修改一下上面的Shell脚本，测试一下yunyu用户是否可以直接使用docker命令

```
echo "execute shell"
echo "jenkins WORKSPACE:"$WORKSPACE
currentUser=`whoami`
echo "currentUser:"$currentUser

# 测试Java的可用性
java -version

# 测试Maven的可用性
mvn -version

# 测试Docker的可用性
docker version
```

控制台日志

```
Started by user birdben
Building in workspace /Users/yunyu/.jenkins/workspace/birdben
[birdben] $ /bin/sh -xe /var/folders/0h/jtjrr7g95mv2pt4ts1tgmzyh0000gn/T/hudson2943867311985687534.sh
+ echo 'execute shell'
execute shell
+ echo 'jenkins WORKSPACE:/Users/yunyu/.jenkins/workspace/birdben'
jenkins WORKSPACE:/Users/yunyu/.jenkins/workspace/birdben
++ whoami
+ currentUser=yunyu
+ echo currentUser:yunyu
currentUser:yunyu
+ java -version
java version "1.7.0_79"
Java(TM) SE Runtime Environment (build 1.7.0_79-b15)
Java HotSpot(TM) 64-Bit Server VM (build 24.79-b02, mixed mode)
+ mvn -version
Apache Maven 3.3.9 (bb52d8502b132ec0a5a3f4c09453c07478323dc5; 2015-11-11T00:41:47+08:00)
Maven home: /Users/yunyu/apache-maven-3.3.9
Java version: 1.7.0_79, vendor: Oracle Corporation
Java home: /Library/Java/JavaVirtualMachines/jdk1.7.0_79.jdk/Contents/Home/jre
Default locale: zh_CN, platform encoding: UTF-8
OS name: "mac os x", version: "10.11.5", arch: "x86_64", family: "mac"
+ docker version
Client:
 Version:      1.12.1
 API version:  1.24
 Go version:   go1.7.1
 Git commit:   6f9534c
 Built:        Thu Sep  8 10:31:18 2016
 OS/Arch:      darwin/amd64

Server:
 Version:      1.12.1
 API version:  1.24
 Go version:   go1.6.3
 Git commit:   23cf638
 Built:        Thu Aug 18 17:52:38 2016
 OS/Arch:      linux/amd64
Finished: SUCCESS
```

#### Ubuntu环境

因为我本机是Mac环境，但是我们公司的测试服务器是Ubuntu环境，所以也在Ubuntu环境下尝试了jenkins用户使用docker命令，但是发现报错如下

```
Cannot connect to the Docker daemon. Is the docker daemon running on this host?
```

这个错误就说明Docker服务没有启动，或者当前用户没有运行docker命令的权限，需要给当前jenkins用户添加到docker用户组才可以，而且一定要重启Jenkins服务。（之前我就是因为没重启Jenkins服务，导致一直误以为将jenkins用户添加到docker用户组也不好用）

### 总结

Jenknis用于自动化构建部署还是很方便的，而且可以自己编写Shell十分灵活，也容易维护。目前公司测试环境使用Jenkins部署Docker十分方便，可以同时部署多个Docker容器，而且不需要自己运行Shell脚本，哈哈

参考文章：

- https://jenkins.io/doc/book/getting-started/installing/
- http://www.cnblogs.com/Leo_wl/p/4314792.html
- http://blog.163.com/bobile45@126/blog/static/9606199220162114956125/