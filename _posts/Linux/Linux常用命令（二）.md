---
title: "Linux常用命令（二）"
date: 2016-06-25 13:38:11
tags: [Linux命令]
categories: [Linux]
---

### scp命令
```
# 获取远程服务器上的文件
$ scp -P 2222 root@192.168.1.100:/root/tomcat.tar.gz /home/tomcat.tar.gz

# -P是端口号参数，2222表示更改SSH端口后的端口，如果没有更改SSH端口可以不用添加该参数 
# root@192.168.1.100 表示使用root用户登录远程服务器192.168.1.100
# :/root/tomcat.tar.gz 表示远程服务器上的文件
# /home/tomcat.tar.gz 表示保存在本地上的路径和文件名

# 获取远程服务器上的目录
$ scp -P 2222 -r root@192.168.1.100:/root/tomcat/ /home/ben/dev/

# -P是端口号参数，2222表示更改SSH端口后的端口，如果没有更改SSH端口可以不用添加该参数
# -r参数表示递归复制（即复制该目录下面的文件和目录）
# root@192.168.1.100 表示使用root用户登录远程服务器192.168.1.100
# :/root/tomcat/ 表示远程服务器上的目录
# /home/ben/dev/ 表示保存在本地上的路径。

# 将本地文件上传到服务器上
$ scp -P 2222 /home/tomcat.tar.gz root@192.168.1.100:/root/tomcat.tar.gz

# -P是端口号参数，2222表示更改SSH端口后的端口，如果没有更改SSH端口可以不用添加该参数。 
# /home/tomcat.tar.gz 表示本地上准备上传文件的路径和文件名
# root@192.168.1.100 表示使用root用户登录远程服务器

# 将本地目录上传到服务器上
$ scp -P 2222 -r /home/tomcat/ root@192.168.1.100:/root/dev/tomcat/

# -P是端口号参数，2222表示更改SSH端口后的端口，如果没有更改SSH端口可以不用添加该参数。
# -r 参数表示递归复制（即复制该目录下面的文件和目录）
# /home/tomcat/表示准备要上传的目录
# root@192.168.1.100 表示使用root用户登录远程服务器192.168.1.100
# :/root/dev/tomcat/ 表示保存在远程服务器上的目录位置。
```


### nc命令

```
# 今天学到了另一种远程传输文件的方式，原来只知道scp，rz/sz
# 远程服务器（目的主机监听）
# 命令格式：nc -l 监听端口<未使用端口> > 要接收的文件名
$ nc -l 9999 > elasticsearch-jdbc-1.7.3.0-dist.zip
# 解释：远程服务器开启一个9999的端口进行监听，并且等待接收elasticsearch-jdbc-1.7.3.0-dist.zip文件

# 本地机器（源主机发起请求）
# 命令格式：nc 目的主机ip 目的端口 < 要发送的文件
$ nc 10.120.10.105 9999 < /Users/ben/Downloads/letv/ES/elasticsearch-jdbc-1.7.3.0-dist.zip
# 解释：本地机器指定远程服务器的ip地址和port端口号将指定的本地文件传送到远程服务器

```

### rz/sz命令

##### Zmodem（rz/sz命令）一般需要和SSH客户端一起配合使用
下面介绍两种客户端

1. iterm2（适用Mac环境）
2. secureCRT（适用Windows环境）

#### iterm2
```
1. 安装brew工具
$ brew install lszrz
# 如果没有brew工具，请参考：http://blog.csdn.net/tsxw24/article/details/15500517
2. 下载iterm2-zmodem到/usr/local/bin目录（将两个shell脚本放到/usr/local/bin/目录下然后开始配置）
$ sudo wget https://raw.github.com/mmastrac/iterm2-zmodem/master/iterm2-send-zmodem.sh 
$ sudo wget https://raw.github.com/mmastrac/iterm2-zmodem/master/iterm2-recv-zmodem.sh 
$ sudo chmod 777 /usr/local/bin/iterm2-*
3. 运行下载的iterm，添加trigger
打开iterm2 -->  同时按command和,键 --> Profiles --> Default --> Advanced --> Triggers的Edit按钮
4. Triggers的设置代码如下 
Regular expression: \*\*B0100
Action: Run Silent Coprocess 
Parameters: /usr/local/bin/iterm2-send-zmodem.sh

Regular expression: \*\*B00000000000000
Action: Run Silent Coprocess 
Parameters: /usr/local/bin/iterm2-recv-zmodem.sh
5. 上传和下载文件和secureCRT方式一样
```
![iterm2](http://img.blog.csdn.net/20160622214939271?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

#### secureCRT
```
# 上传文件
1. 打开secureCRT,通过SSH连到至远程linux主机
2. 键入rz命令
3. 在跳出的窗口选择想要上传的文件
4. 点击ADD后加入传输列表
5. 点击确认以后传送文件
6. 查看状态，一般都是发送正功
7. 查看目录，确定刚才选的文件已经上传成功

# 下载文件
1. 远程连接到linux主机
2. 输入sz fileName
3. 在弹出的窗口中选择保存文件到本地的目录
4. 确定后，等待文件下载完成
```

参考文章：

- http://www.vpser.net/manage/scp.html
- http://blog.csdn.net/u014552288/article/details/38349827
- http://blog.csdn.net/tsxw24/article/details/15500517
- http://wenku.baidu.com/link?url=QV3BHlMEVRd7c2Nlw8p4_o9z6qXecNoGACOLLGdYxP3qnM8XIujaUPk4ZZZl83Docq4fxxl3BMRijuVAy4yryGVdlFB1jM9JDzGs5L5eHuO
- http://jingyan.baidu.com/article/b24f6c822bdabc86bee5da64.html