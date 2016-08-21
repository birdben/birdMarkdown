---
title: "Docker实战（十）Docker安装Nginx"
date: 2016-01-02 17:43:04
tags: [Docker命令, Dockerfile, Nginx]
categories: [Docker]
---

安装Nginx可以选择直接使用ubuntu的apt-get install nginx命令来安装，这种安装方式最简单方便，但是Nginx的版本可能是比较老的版本，所以这里我选择编译安装的方式。##### Nginx需要依赖下面3个包

1. gzip 模块需要 zlib 库 ( 下载: http://www.zlib.net/ )  zlib-1.2.8.tar.gz
2. rewrite 模块需要 pcre 库 ( 下载: http://www.pcre.org/ )  pcre-8.37.tar.gz
3. ssl 功能需要 openssl 库 ( 下载: http://www.openssl.org/ )  openssl-1.0.1q.tar.gz##### 编译方式安装Nginx```
# 下载nginx安装依赖的包
$ wget http://zlib.net/zlib-1.2.8.tar.gz
$ wget http://www.openssl.org/source/openssl-1.0.1q.tar.gz
$ wget http://downloads.sourceforge.net/project/pcre/pcre/8.37/pcre-8.37.tar.gz
$ wget http://nginx.org/download/nginx-1.8.0.tar.gz

# nginx-1.8.0，pcre-8.37，zlib-1.2.8，openssl-1.0.1q这几个解压的文件夹是放在/temp文件夹下，nginx按照的目录是/software/nginx-1.8.0$ cd /temp/nginx-1.8.0/$ sudo ./configure --sbin-path=/software/nginx-1.8.0/nginx --conf-path=/software/nginx-1.8.0/nginx.conf --pid-path=/software/nginx-1.8.0/nginx.pid --with-http_ssl_module --with-pcre=/temp/pcre-8.37 --with-zlib=/temp/zlib-1.2.8 --with-openssl=/temp/openssl-1.0.1q
$ sudo make
$ sudo make install

# 检查80端口是否被占用
$ netstat -ano|grep 80

# 启动nginx
$ cd /software/nginx-1.8.0/
$ sudo ./nginx

# 不指定配置文件地址
$ cd /software/nginx-1.8.0
$ ./nginx

# 指定配置文件地址
$ cd /software/nginx-1.8.0
$ ./nginx -c /software/nginx-1.8.0/nginx.conf

# 停止服务
$ sudo kill 'cat /software/nginx-1.8.0/nginx.pid'

# 检测配置文件
$ cd /software/nginx-1.8.0
$ ./nginx -t

# 重新加载配置文件(不停止服务)
$ cd /software/nginx-1.8.0
$ ./nginx -s reload
```
##### 编译安装Nginx可能会遇到的问题和解决方法

```
# 编译安装Nginx的时候需要指定相关参数，否则会遇到下面的问题
./configure: error: the HTTP rewrite module requires the PCRE library.You can either disable the module by using --without-http_rewrite_moduleoption, or install the PCRE library into the system, or build the PCRE librarystatically from the source with nginx by using --with-pcre=<path> option.

# Nginx编译参数
–prefix 						#nginx安装目录，默认在/usr/local/nginx
–pid-path 						#pid问件位置，默认在logs目录
–lock-path 						#lock问件位置，默认在logs目录
–with-http_ssl_module 			#开启HTTP SSL模块，以支持HTTPS请求。
–with-http_dav_module 			#开启WebDAV扩展动作模块，可为文件和目录指定权限
–with-http_flv_module 			#支持对FLV文件的拖动播放
–with-http_realip_module 		#支持显示真实来源IP地址
–with-http_gzip_static_module 	#预压缩文件传前检查，防止文件被重复压缩
–with-http_stub_status_module 	#取得一些nginx的运行状态
–with-mail 						#允许POP3/IMAP4/SMTP代理模块
–with-mail_ssl_module 			#允许POP3／IMAP／SMTP可以使用SSL／TLS
–with-pcre=../pcre-8.11 		#注意是未安装的pcre路径
–with-zlib=../zlib-1.2.5 		#注意是未安装的zlib路径
–with-debug 					#允许调试日志
–http-client-body-temp-path 	#客户端请求临时文件路径
–http-proxy-temp-path 			#设置http proxy临时文件路径
–http-fastcgi-temp-path 		#设置http fastcgi临时文件路径
–http-uwsgi-temp-path=/var/tmp/nginx/uwsgi #设置uwsgi 临时文件路径
–http-scgi-temp-path=/var/tmp/nginx/scgi #设置scgi 临时文件路径# 我用pcre2替代了pcre会遇到如下的问题，最后还是换成pcre
出现了错误：src/core/ngx_regex.h:15:18: fatal error: pcre.h: No such file or directory #include <pcre.h>                  ^compilation terminated.make[1]: *** [objs/src/core/nginx.o] Error 1make[1]: Leaving directory `/home/ben/dev/nginx-1.8.0'make: *** [build] Error 2


# 如果遇到下面的问题最好升级一下gcc
$ sudo rm -rf /var/lib/apt/lists/*
$ apt-get update

configure: error: You need a C++ compiler for C++ support.make[1]: *** [/home/ben/dev/pcre-8.37/Makefile] Error 1make[1]: Leaving directory `/home/ben/dev/nginx-1.8.0'make: *** [build] Error 2```

##### Dockerfile文件

```
############################################
# version : birdben/nginx:v1
# desc : 当前版本安装的nginx
############################################
# 设置继承自我们创建的 tools 镜像
FROM birdben/tools:v1

# 下面是一些创建者的基本信息
MAINTAINER birdben (191654006@163.com)

# 设置环境变量，所有操作都是非交互式的
ENV DEBIAN_FRONTEND noninteractive

# 添加 supervisord 的配置文件，并复制配置文件到对应目录下面。（supervisord.conf文件和Dockerfile文件在同一路径）
COPY supervisord.conf /etc/supervisor/conf.d/supervisord.conf

# 安装升级gcc
RUN sudo rm -rf /var/lib/apt/lists/*
RUN sudo apt-get update

RUN sudo apt-get -y install \
build-essential

RUN sudo mkdir -p /software/temp
RUN wget http://nginx.org/download/nginx-1.8.0.tar.gz && tar -zxvf nginx-1.8.0.tar.gz -C /software/temp
RUN wget http://zlib.net/zlib-1.2.8.tar.gz && tar -zxvf zlib-1.2.8.tar.gz -C /software/temp
RUN wget http://www.openssl.org/source/openssl-1.0.1q.tar.gz && tar -zxvf openssl-1.0.1q.tar.gz -C /software/temp
RUN wget http://downloads.sourceforge.net/project/pcre/pcre/8.37/pcre-8.37.tar.gz && tar -zxvf pcre-8.37.tar.gz -C /software/temp
RUN cd /software/temp/nginx-1.8.0 && sudo ./configure --sbin-path=/software/nginx-1.8.0/nginx --conf-path=/software/nginx-1.8.0/nginx.conf --pid-path=/software/nginx-1.8.0/nginx.pid --with-http_ssl_module --with-pcre=/software/temp/pcre-8.37 --with-zlib=/software/temp/zlib-1.2.8 --with-openssl=/software/temp/openssl-1.0.1q && sudo make && sudo make install

# 设置nginx是非daemon启动
RUN echo 'daemon off;' >> /software/nginx-1.8.0/nginx.conf
RUN echo 'master_process off;' >> /software/nginx-1.8.0/nginx.conf
RUN echo 'error_log  logs/error.log;' >> /software/nginx-1.8.0/nginx.conf

# 设置 NGINX 的环境变量，若读者有其他的环境变量需要设置，也可以在这里添加。
ENV NGINX_HOME /software/nginx-1.8.0

# 容器需要开放Nginx 80端口
EXPOSE 80

# 执行run.sh文件
# CMD ["/run.sh"]
# 执行supervisord来同时执行多个命令，使用 supervisord 的可执行路径启动服务。
CMD ["/usr/bin/supervisord"]
```

##### Dockerfile源文件链接：

https://github.com/birdben/birdDocker/blob/master/nginx/Dockerfile

##### 注意
```
# 编译安装Nginx方式构建Docker镜像遇到几个比较坑的问题
1. supervisor无法启动Nginx，必须设置nginx是非daemon启动，这个问题之前安装ES的时候已经遇到过了，因为没有注意还是被这个问题困扰了一段时间。Nginx设置非daemon启动需要在nginx.conf配置文件中设置daemon false即可。
参考文章：
http://comments.gmane.org/gmane.comp.sysutils.supervisor.general/1233
http://nginx.org/en/docs/ngx_core_module.html#daemon
http://serverfault.com/questions/647357/running-and-monitoring-nginx-with-supervisord
http://www.v2ex.com/t/123152

2. 编译安装Nginx时，总是提示lib-dev64相对的版本不匹配，即使执行apt-get -y install build-essential安装gcc也是如此，最终发现是因为之前继承的tools镜像在安装ssh时有改过ubuntu的sourcelist文件，直接继承ubuntu 14.04的镜像就不会有lib-dev64版本不匹配的问题。
参考文章：
http://www.oschina.net/question/220489_160699
http://www.cnblogs.com/linxiong945/p/4180565.html
http://blog.csdn.net/xhz1234/article/details/37044531
```

##### Supervisor.conf

```
# 配置文件包含目录和进程
# 第一段 supervsord 配置软件本身，使用 nodaemon 参数来运行。
# 第二段包含要控制的 2 个服务。每一段包含一个服务的目录和启动这个服务的命令。

[supervisord]
nodaemon=true

[program:sshd]
command=/usr/sbin/sshd -D

[program:nginx]
command=/software/nginx-1.8.0/nginx -c /software/nginx-1.8.0/nginx.conf
```

##### 控制台终端
```
# 构建镜像
docker build -t="birdben/nginx:v1" .
# 执行已经构件好的镜像
docker run -p 9999:22 -p 8888:80 -t -i birdben/nginx:v1
```

##### 浏览器访问
```
http://10.211.55.4:8888/```

参考文章

- http://nginx.org/en/docs/configure.html
- http://www.cnblogs.com/skynet/p/4146083.html
- http://www.nginx.cn/install
- http://www.cnblogs.com/siqi/p/3572695.html
- https://www.nginx.com/resources/wiki/start/topics/tutorials/installoptions/