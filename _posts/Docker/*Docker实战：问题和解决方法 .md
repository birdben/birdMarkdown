## Docker学习遇到的问题和解决方法

#### 1. Docker构建镜像过程中，无法安装ssh，甚至无法执行修改ssh密码的命令

这个问题实在是令我好纠结啊，周末为了排查这个问题找了很久也没找到原因，在网上很多的文章也发现有的人可以执行成功，有的人不能执行成功。开始以为是pull下来的ubuntu:14.04镜像不是官方的，后来验证没有问题。后来又猜想是宿主机的ubuntu版本不对，或者不是64位的，验证过后都没有问题。

```
ben@ubuntu:~$ cat /etc/issueUbuntu 14.04.2 LTS \n \l

ben@ubuntu:~$ lsb_release -aNo LSB modules are available.Distributor ID:	UbuntuDescription:	Ubuntu 14.04.2 LTSRelease:	14.04Codename:	trusty

ben@ubuntu:~$ sudo uname -mx86_64
```

最后我想到了一个方式，在DaoCloud中尝试执行我自己的Dockerfile来构建我的镜像，发现DaoCloud安装的docker的版本与我本地安装的docker版本不一致导致的，我本地安装的docker版本太低了，升级之后问题就解决了，安装和链接ssh都没有问题，也可以在Dockerfile中修改ssh密码了。

升级的方法：（从Docker官方源安装最新的版本，首先需要安装apt-transport-https，并添加Docker官方源）

```
ben@ubuntu:~$ sudo apt-get install apt-transport-https  
# Add the Docker repository key to your local keychain  
ben@ubuntu:~$ sudo apt-key adv --keyserver keyserver.ubuntu.com --recv-keys 36A1D7869245C8950F966E92D8576A8BA88D21E9  
# Add the Docker repository to your apt sources list.  
ben@ubuntu:~$ sudo sh -c "echo deb https://get.docker.io/ubuntu docker main > /etc/apt/sources.list.d/docker.list"  
# update your sources list  
ben@ubuntu:~$ sudo apt-get update  
   
# 之后通过下面命令来安装最新版本的docker：  
ben@ubuntu:~$ sudo apt-get install -y lxc-docker  
# 以后更新则：  
ben@ubuntu:~$ sudo apt-get update -y lxc-docker  
  
ben@ubuntu:~$ ln -sf /usr/bin/docker /usr/local/bin/docker  
```

```
# 更新的过程中如果遇到包依赖错误，执行下面语句后重新更新
ben@ubuntu:~$ sudo rm -rf /var/lib/apt/lists/*
ben@ubuntu:~$ sudo rm -rf 具体的依赖冲突的包路径
```

DaoCloud中的Docker client版本如下：

```
ubuntu@ip-10-23-141-9:~$ sudo docker version
Client version: 1.7.0
Client API version: 1.19
Go version (client): go1.4.2
Git commit (client): 0baf609
OS/Arch (client): linux/amd64
Server version: 1.7.0
Server API version: 1.19
Go version (server): go1.4.2
Git commit (server): 0baf609
OS/Arch (server): linux/amd64
```

我本地升级docker之后的版本如下：（升级之前的版本忘记保存了。。使用ubuntu默认apt-get install命令直接安装的，好像是1.0.1版本）

```
ben@ubuntu:~$ sudo docker versionClient: Version:      1.9.0 API version:  1.21 Go version:   go1.4.3 Git commit:   76d6bc9 Built:        Tue Nov  3 19:20:09 UTC 2015 OS/Arch:      linux/amd64Server: Version:      1.9.0 API version:  1.21 Go version:   go1.4.3 Git commit:   76d6bc9 Built:        Tue Nov  3 19:20:09 UTC 2015 OS/Arch:      linux/amd64```

参考文章：

- http://blog.csdn.net/delphiwcdj/article/details/42836423

#### 2. Docker强制删除none的image镜像

docker中会出现none的镜像，是因为在build的过程中报错了，所以会出现一个none的镜像

##### 查看docker中的所有镜像

`ben@ubuntu:~/dev$ docker images`
```sequence
|REPOSITORY  |TAG      |IMAGE ID      |CREATED          |VIRTUAL SIZE|
|------------|---------|--------------|-----------------|------------|
|none        |none     |262ce37fcc3c  |17 minutes ago   |212.7 MB    ||ubuntu      |14.04    |a5a467fddcb8  |5 days ago       |187.9 MB    |
``` 

##### 查看当前docker中运行的容器
`ben@ubuntu:~/dev$ docker ps -a````sequence|CONTAINER ID  |IMAGE         |COMMAND                |CREATED        |STATUS                       |PORTS  | NAMES                 |
|--------------|--------------|-----------------------|---------------|-----------------------------|-------|-----------------------|
|d1d9adbabb39  |af89d7642666  |"/bin/sh -c 'echo "r   |15 minutes ago |Exited (1) 15 minutes ago    |       |nostalgic_goldstine    |  
|f166d94d8fb2  |ubuntu:14.04  |/bin/bash              |29 hours ago   |Exited (0) 27 hours ago      |       |focused_morse          |
``` 

##### 停止docker容器（容器ID）
`ben@ubuntu:~/dev$ docker stop d1d9adbabb39`

##### 删除docker容器（容器ID）
`ben@ubuntu:~/dev$ docker rm af89d7642666`

##### 删除docker镜像（镜像ID）
`ben@ubuntu:~/dev$ docker rmi 262ce37fcc3c`