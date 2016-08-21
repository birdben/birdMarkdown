---
title: "Docker实战（十三）Docker安装MySQL数据库"
date: 2016-06-19 15:20:57
tags: [Docker命令, Dockerfile, MySQL]
categories: [Docker]
---

基本步骤和之前几篇文章一样，请参考前面的相关文章

##### Ubuntu安装MySQL安装

```
（1）安装编译源码需要的包
sudo apt-get install make cmake gcc g++ bison libncurses5-dev build-essential

（2）下载并解压缩
mysql-5.6.26.tar.gz
tar -zxvf mysql-5.6.26.tar.gz
cd mysql-5.6.26

（3）编译安装
编译配置：
cmake . 
-DCMAKE_INSTALL_PREFIX=/usr/local/mysql 
-DMYSQL_DATADIR=/usr/local/mysql/data 
-DSYSCONFDIR=/etc 
-DWITH_INNOBASE_STORAGE_ENGINE=1 
-DWITH_ARCHIVE_STORAGE_ENGINE=1 
-DWITH_BLACKHOLE_STORAGE_ENGINE=1 
-DWITH_PARTITION_STORAGE_ENGINE=1 
-DWITH_PERFSCHEMA_STORAGE_ENGINE=1 
-DWITHOUT_EXAMPLE_STORAGE_ENGINE=1 
-DWITHOUT_FEDERATED_STORAGE_ENGINE=1 
-DDEFAULT_CHARSET=utf8 
-DDEFAULT_COLLATION=utf8_general_ci 
-DWITH_EXTRA_CHARSETS=all 
-DENABLED_LOCAL_INFILE=1 
-DWITH_READLINE=1 
-DMYSQL_UNIX_ADDR=/usr/local/mysql/mysql.sock 
-DMYSQL_TCP_PORT=3306 
-DMYSQL_USER=mysql 
-DCOMPILATION_COMMENT="lq-edition"
-DENABLE_DTRACE=0 
-DOPTIMIZER_TRACE=1 
-DWITH_DEBUG=1

cmake . -DCMAKE_INSTALL_PREFIX=/usr/local/mysql -DMYSQL_DATADIR=/usr/local/mysql/data -DSYSCONFDIR=/etc -DWITH_INNOBASE_STORAGE_ENGINE=1 -DWITH_ARCHIVE_STORAGE_ENGINE=1 -DWITH_BLACKHOLE_STORAGE_ENGINE=1 -DWITH_PARTITION_STORAGE_ENGINE=1 -DWITH_PERFSCHEMA_STORAGE_ENGINE=1 -DWITHOUT_EXAMPLE_STORAGE_ENGINE=1 -DWITHOUT_FEDERATED_STORAGE_ENGINE=1 -DDEFAULT_CHARSET=utf8 -DDEFAULT_COLLATION=utf8_general_ci -DWITH_EXTRA_CHARSETS=all -DENABLED_LOCAL_INFILE=1 -DWITH_READLINE=1 -DMYSQL_UNIX_ADDR=/usr/local/mysql/mysql.sock -DMYSQL_TCP_PORT=3306 -DMYSQL_USER=mysql -DCOMPILATION_COMMENT="lq-edition" -DENABLE_DTRACE=0 -DOPTIMIZER_TRACE=1 -DWITH_DEBUG=1

参数说明：

-DCMAKE_INSTALL_PREFIX=/usr/local/mysql          //安装目录
-DINSTALL_DATADIR=/usr/local/mysql/data          //数据库存放目录
-DDEFAULT_CHARSET=utf8                    　　　　//使用utf8字符
-DDEFAULT_COLLATION=utf8_general_ci              //校验字符
-DWITH_EXTRA_CHARSETS=all                        //安装所有扩展字符集
-DENABLED_LOCAL_INFILE=1                    　　  //允许从本地导入数据

编译：
make

安装：
sudo make install

配置MySQL
（1）新建运行Mysql的用户和组
sudo groupadd mysql
sudo useradd -g mysql mysql

登录mysql用户
先修改mysql用户的密码
passwd mysql

（2）设置Mysql安装目录的权限
cd /usr/local/mysql
sudo chown -R mysql:mysql ./

（3）建立配置文件
cp support-files/my-default.cnf /etc/my.cnf
sudo chown mysql:mysql /etc/my.cnf
修改配置文件：
sudo vi /etc/my.cnf
[client]
port = 3306
socket = /usr/local/mysql/data/mysql.sock
[mysqld]
port = 3306
socket = /usr/local/mysql/data/mysql.sock
basedir = /usr/local/mysql
datadir  = /usr/local/mysql/data

（4）初始化数据库
cd /usr/local/mysql
sudo ./scripts/mysql_install_db --user=mysql --basedir=/usr/local/mysql --datadir=/usr/local/mysql/data/

（5）启动mysql服务
方法1：
直接启动
bin/mysqld_safe &
检查MySQL服务是否启动：
ps -ef |grep mysql

（6）配置环境变量
为了直接调用mysql，需要将mysql的bin目录加入PATH环境变量。
编辑/etc/profile文件：
sudo vim /etc/profile
在文件最后 添加如下两行：
PATH=$PATH:/usr/local/mysql/bin
export PATH
关闭文件，运行下面的命令，让配置立即生效：
source /etc/profile

（7）修改root密码（因为默认密码为空）
$ mysql -u root -p
mysql> UPDATE user SET Password=PASSWORD('你想要的密码') where USER='root';
mysql> FLUSH PRIVILEGES;

（8）授权root账户远程登录
$ mysql -u root -p
mysql> grant ALL PRIVILEGES ON *.* to root@"%" identified by "root" WITH GRANT OPTION;
mysql> FLUSH PRIVILEGES;  

```
##### Dockerfile文件
```
############################################
# version : birdben/ubuntu:mysql
# desc : 当前版本安装的MySQL
############################################
# 设置继承自我们创建的 tools 镜像
FROM birdben/ubuntu:tools

# 下面是一些创建者的基本信息
MAINTAINER birdben (191654006@163.com)

# 设置环境变量，所有操作都是非交互式的
ENV DEBIAN_FRONTEND noninteractive

# 添加 supervisord 的配置文件，并复制配置文件到对应目录下面。（supervisord.conf文件和Dockerfile文件在同一路径）
COPY supervisord.conf /etc/supervisor/conf.d/supervisord.conf

RUN echo "export LC_ALL=C"

# 替换ubuntu软件更新的源服务器的sources.list文件
COPY sources.list /etc/apt/sources.list

# 安装升级gcc
RUN sudo rm -rf /var/lib/apt/lists/*
RUN sudo apt-get update
RUN sudo apt-get install -y make cmake gcc g++ bison libncurses5-dev build-essential

# 复制 mysql-5.6.22 文件到镜像中（mysql-5.6.22文件夹要和Dockerfile文件在同一路径）
ADD mysql-5.6.22 /software/downloads/mysql-5.6.22

RUN cd /software/downloads/mysql-5.6.22 && cmake . -DCMAKE_INSTALL_PREFIX=/software/mysql-5.6.22 -DMYSQL_DATADIR=/software/mysql-5.6.22/data -DSYSCONFDIR=/etc -DWITH_INNOBASE_STORAGE_ENGINE=1 -DWITH_ARCHIVE_STORAGE_ENGINE=1 -DWITH_BLACKHOLE_STORAGE_ENGINE=1 -DWITH_PARTITION_STORAGE_ENGINE=1 -DWITH_PERFSCHEMA_STORAGE_ENGINE=1 -DWITHOUT_EXAMPLE_STORAGE_ENGINE=1 -DWITHOUT_FEDERATED_STORAGE_ENGINE=1 -DDEFAULT_CHARSET=utf8 -DDEFAULT_COLLATION=utf8_general_ci -DWITH_EXTRA_CHARSETS=all -DENABLED_LOCAL_INFILE=1 -DWITH_READLINE=1 -DMYSQL_UNIX_ADDR=/software/mysql-5.6.22/mysql.sock -DMYSQL_TCP_PORT=3306 -DMYSQL_USER=mysql -DCOMPILATION_COMMENT="lq-edition" -DENABLE_DTRACE=0 -DOPTIMIZER_TRACE=1 -DWITH_DEBUG=1 && make && make install

# 添加测试用户mysql，密码mysql，并且将此用户添加到sudoers里
RUN useradd mysql
RUN echo "mysql:mysql" | chpasswd
RUN echo "mysql   ALL=(ALL)       ALL" >> /etc/sudoers

# 设置Mysql安装目录的权限
RUN cd /software/mysql-5.6.22 && sudo chown -R mysql:mysql ./

# 复制已经准备好的my.cnf文件到Docker容器
COPY my.cnf /etc/my.cnf
RUN sudo chown mysql:mysql /etc/my.cnf

# 初始化数据库
RUN cd /software/mysql-5.6.22 && sudo ./scripts/mysql_install_db --user=mysql --basedir=/software/mysql-5.6.22 --datadir=/software/mysql-5.6.22/data/

# 设置MySQL的环境变量，若读者有其他的环境变量需要设置，也可以在这里添加。
ENV MYSQL_HOME /software/mysql-5.6.22

# （不推荐下面的路径直接建立在Docker虚拟机上，推荐使用volume挂载方式）
# 在宿主机上创建一个数据库目录存储Mysql的数据文件
# sudo mkdir -p /docker/mysql/data

# VOLUME 选项是将本地的目录挂在到容器中　此处要注意：当你运行-v　＜hostdir>:<Containerdir> 时要确保目录内容相同否则会出现数据丢失
# 对应关系如下
# mysql:/docker/mysql/data
VOLUME ["/software/mysql-5.6.22/data"]

# 容器需要开放MySQL 3306端口
EXPOSE 3306

# 执行supervisord来同时执行多个命令，使用 supervisord 的可执行路径启动服务。
CMD ["/usr/bin/supervisord"]```

##### Dockerfile源文件链接：

https://github.com/birdben/birdDocker/blob/master/mysql/Dockerfile

##### supervisor配置文件内容

```
# 配置文件包含目录和进程
# 第一段 supervsord 配置软件本身，使用 nodaemon 参数来运行。
# 第二段包含要控制的 2 个服务。每一段包含一个服务的目录和启动这个服务的命令。

[supervisord]
nodaemon=true

[program:sshd]
command=/usr/sbin/sshd -D

[program:mysqld]
# 这里使用mysqld_safe &方式启动MySQL服务，但是不知道为什么后面加上&启动MySQL会报错，不加&就能正常启动了，后来看了一遍自己以前安装ES的博客才发现，mysql启动必须改成非daemon启动方式，这样supervisor就可以监控到了，所以要去掉&
command=./software/mysql-5.6.22/bin/mysqld_safe --user=mysql
```

##### my.cnf（复制MySQL_HOME/support-files/my-default.cnf文件修改如下）

```
[client]
port = 3306
socket = /software/mysql-5.6.22/data/mysql.sock
[mysqld]
port = 3306
socket = /software/mysql-5.6.22/data/mysql.sock
basedir = /software/mysql-5.6.22
datadir  = /software/mysql-5.6.22/data
```


##### 控制台终端
```
# 构建镜像
$ docker build -t="birdben/mysql:v1" .
# 执行已经构件好的镜像
$ docker run -p 9999:22 -p 3306:3306 -t -i -v /docker/mysql/data:/software/mysql-5.6.22/data birdben/mysql:v1


# 可以ssh远程登录，然后登录mysql就大功告成了
$ ssh admin@10.211.55.4 -p 9999
# root账号的默认密码是空
$ cd /software/mysql-5.6.22/bin
$ mysql -u root -p

# 修改root密码（因为默认密码为空）
mysql> use mysql;
mysql> UPDATE user SET Password=PASSWORD('你想要的密码') where USER='root';
mysql> FLUSH PRIVILEGES;

# 授权root账户远程登录，然后就可以通过navicat或者其他客户端工具连接到MySQL服务器了
mysql> grant ALL PRIVILEGES ON *.* to root@"%" identified by "root" WITH GRANT OPTION;
mysql> FLUSH PRIVILEGES; 


```

参考文章

- http://www.linuxdiyf.com/linux/14453.html
- http://linux.it.net.cn/e/data/mysql/2016/0103/19536.html
- http://blog.itpub.net/29071259/viewspace-1368508/

