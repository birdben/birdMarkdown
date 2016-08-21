---
title: "Shell脚本学习（二）制作SSH登录远程服务器的Shell脚本"
date: 2016-08-09 22:58:55
tags: [Shell]
categories: [Shell]
---

Ubuntu环境需要安装expect安装包

```
sudo apt-get install expect
```

使用shell脚本自动ssh登录远程服务器

login.sh

```
#!/usr/bin/expect -f
# 设置ssh连接的用户名
set user liuben
# 设置ssh连接的host地址
set host 10.211.55.4
# 设置ssh连接的port端口号
set port 9999
# 设置ssh连接的登录密码
set password admin
# 设置ssh连接的超时时间
set timeout -1

spawn ssh $user@$host -p $port
expect "*password:"
# 提交密码
send "$password\r"
# 控制权移交
interact
```

```
# 确定login.sh脚本有可执行权限
chmod +x login.sh
# 执行login.sh脚本
./login.sh

# 注意
不能按照习惯来用sh login.sh来这行expect的程序，会提示找不到命令，如下：

login.sh: line 3: spawn: command not found
couldn't read file "*password:": no such file or directory
login.sh: line 5: send: command not found
login.sh: line 6: interact: command not found

因为expect用的不是bash所以会报错。因为bash和expect的脚本指定了不同的脚本解释器
#!/usr/bin/expect -f
#!/bin/bash
执行的时候直接./login.sh就可以了。～切记！
```

参考文章：

- http://blog.csdn.net/zhuying_linux/article/details/6657020