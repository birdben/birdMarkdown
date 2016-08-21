---
title: "Linux常用命令（一）"
date: 2015-12-09 21:32:01
tags: [Linux命令]
categories: [Linux]
---

### netstat查看端口
```
# 参数
-a (all)显示所有选项，默认不显示LISTEN相关
-t (tcp)仅显示tcp相关选项
-u (udp)仅显示udp相关选项
-n 拒绝显示别名，能显示数字的全部转化成数字。将应用程序转为端口显示，即数字格式的地址，如：nfs->2049，ftp->21，因此可以开启两个终端，一一对应一下程序所对应的端口号）
-l 仅列出有在 Listen (监听) 的服務状态

-p 显示建立相关链接的程序名
-r 显示路由信息，路由表
-e 显示扩展信息，例如uid等
-s 按各个协议进行统计
-c 每隔一个固定时间，执行该netstat命令。

提示：LISTEN和LISTENING的状态只有用-a或者-l才能看到

# 查看Linux端口号
$ netstat -anp | grep 80

# 查看服务对应端口
$ netstat -nlp
```

### netstat查看并发连接数

```
netstat -n | awk '/^tcp/ {++S[$NF]} END {for(a in S) print a, S[a]}'
返回结果示例：
　　LAST_ACK 5
　　SYN_RECV 30
　　ESTABLISHED 1597
　　FIN_WAIT1 51
　　FIN_WAIT2 504
　　TIME_WAIT 1057

状态：描述
　　CLOSED：无连接是活动的或正在进行
　　LISTEN：服务器在等待进入呼叫
　　SYN_RECV：一个连接请求已经到达，等待确认；正在等待处理的请求数
　　SYN_SENT：应用已经开始，打开一个连接
　　ESTABLISHED：正常数据传输状态
　　FIN_WAIT1：应用说它已经完成
　　FIN_WAIT2：另一边已同意释放
　　ITMED_WAIT：等待所有分组死掉
　　CLOSING：两边同时尝试关闭
　　TIME_WAIT：另一边已初始化一个释放；表示处理完毕，等待超时结束的请求数。
　　LAST_ACK：等待所有分组死掉
```

### lsof查看端口
```
# 查看8080端口号对应的进程
$ lsof -i:8080
```

### ps查看进程
```
# 查看Linux端口号
# ps 命令用于查看当前正在运行的进程
# grep 是搜索
$ ps -ef | grep 80

# 查看Linux进程
$ ps -ef | grep tomcat

# -aux 显示所有状态
$ ps -aux | grep tomcat

# 查看服务进程
$ ps -aux

ps [选项]
-e 显示所有进程,环境变量
-f 全格式
-h 不显示标题
-l 长格式
-w 宽输出
-a 显示终端上地所有进程,包括其他用户地进程
-r 只显示正在运行地进程
-x 显示没有控制终端地进程
```

### Linux性能监控
```
# 查看服务cpu利用
$ top

s - 改变画面更新频率
l - 关闭或开启第一部分第一行 top 信息的表示
t - 关闭或开启第一部分第二行 Tasks 和第三行 Cpus 信息的表示
m - 关闭或开启第一部分第四行 Mem 和 第五行 Swap 信息的表示
N - 以 PID 的大小的顺序排列表示进程列表（第三部分后述）
P - 以 CPU 占用率大小的顺序排列进程列表 （第三部分后述）
M - 以内存占用率大小的顺序排列进程列表 （第三部分后述）
h - 显示帮助
n - 设置在进程列表所显示进程的数量
q - 退出 top

# 查看内存使用情况
$ free

下面是对这些数值的解释：
total:总计物理内存的大小。
used:已使用多大。
free:可用有多少。
Shared:多个进程共享的内存总额。
Buffers/cached:磁盘缓存的大小。

第三行(-/+ buffers/cached):
used:已使用多大。
free:可用有多少。

空闲内存 = free + buffers + cached = total - used

# 显示目前所有文件系统的可用空间及使用情形
$ df -h

# 显示当前目录下所有文件和文件夹的大小
$ ls -lh
```

### tcpdump数据包抓取

```
# tcpdump命令格式
$ tcpdump [-nn] [-i接口] [-w 存储档名] [-c 次数] [-Ae] [-qX] [-r 档案] [所要截取的数据内容]

参数：
-nn：直接以IP及port number显示
-v：输出一个稍微详细的信息，例如在ip包中可以包括ttl和服务类型的信息
-i：后接要监听的网络接口如eth0，wlan0，lo，ppp0
-w：如果你要讲监听所得的封包数据存下来，就用这个参数，后接存档名
-c：监听包个数，若不接会一直监听
-A：封包内容以ASCII码显示，常用语截取www
-q：列出较为尖端的封包信息，每行内容比较精简
-X：可以列出十六进制以及ASCII的封包内容，对于监听封包内容很有用
-r：从后面接的档案封包数据读出来，那个【档案】是已经存在的档案，且这【档案】也是由-w制作出来的


# 抓取包含192.168.1.3的数据包
$ tcpdump -i eth0 -vnn host 192.168.1.3

# 抓取包含192.168.1.3/10网段的数据包
$ tcpdump -i eth0 -vnn net 192.168.1.3/10

# 抓取包含端口22的数据包
$ tcpdump -i eth0 -vnn port 22

# 抓取udp协议的数据包
$ tcpdump -i eth0 -vnn udp

# 抓取ip协议的数据包
$ tcpdump -i eth0 -vnn ip

# 抓取源ip是192.168.1.3数据包
$ tcpdump -i eth0 -vnn src host 192.168.1.3

# 抓取目的ip是211.94.114.21数据包
$ tcpdump -i eth0 -vnn dst host 211.94.114.21

# 抓取源端口是22的数据包
$ tcpdump -i eth0 -vnn src port 22

# 抓取源ip是192.168.1.3且目的端口是22的数据包
$ tcpdump -i eth0 -vnn src host 192.168.1.3 and dst port 22

# 抓取源ip是192.168.1.3或者包含端口是22的数据包
$ tcpdump -i eth0 -vnn src host 192.168.1.3 or port 22

# 抓取源ip是192.168.1.3且端口不是22的数据包
$ tcpdump -i eth0 -vnn src host 192.168.1.3 and not port 22

# 抓取源ip是192.168.1.3且目的端口是22，或源ip是192.168.1.3且目的端口是80的数据包
$ tcpdump -i eth0 -vnn src host 192.168.1.3 and dst port 22 or src host 192.168.1.3 and dst port 80

# 把抓取ip是192.168.1.3的数据包记录存到/Users/ben/Downloads/tcpdump.cap文件中，当抓取100个数据包后就退出程序
$ tcpdump -i eth0 -vnn -w /Users/ben/Downloads/tcpdump.cap -c 100 host 192.168.1.3

# 从/Users/ben/Downloads/tcpdump.cap记录中读取tcp协议的数据包
$ tcpdump -i eth0 -vnn -r /Users/ben/Downloads/tcpdump.cap tcp

# 从/Users/ben/Downloads/tcpdump.cap记录中读取包含192.168.1.3的数据包
$ tcpdump -i eth0 -vnn -r /Users/ben/Downloads/tcpdump.cap host 192.168.1.3

# 抓包结果
00:36:48.658266 IP (tos 0x0, ttl 64, id 15221, offset 0, flags [DF], proto TCP (6), length 40)    10.211.55.4.32903 > 61.135.169.125.443: Flags [.], cksum 0xe7a0 (correct), ack 4785, win 343, length 0

# 抓包结果解释
00:36:48.658266    抓包时间
10.211.55.4.32903  发送端IP以及端口32903
61.135.169.125.443 接收端IP以及端口443
```

### find文件查找相关

```
# find <指定目录> <指定条件> <指定动作>
　　- <指定目录>： 所要搜索的目录及其所有子目录。默认为当前目录。
　　- <指定条件>： 所要搜索的文件的特征。
　　- <指定动作>： 对搜索结果进行特定的处理。

# 搜索当前目录中，所有文件名以my开头的文件 -name（文件名需要带后缀）
$ find . -name 'my*'

# 搜索当前目录中，所有文件名以my开头的文件，并显示它们的详细信息。
$ find . -name 'my*' -ls

# 以文件大小来查找 -size n
$ find . -size +1000000c（在当前目录下查找文件长度大于1 M字节的文件 ）

# 在当前目录及所有子目录中查找filename(忽略大小写)
$ find . -iname "Hessian.properties"
```

### grep文件内容查找相关

```
# 读出logcat.log文件的内容，通过管道转发给grep作为输入内容,过滤包含"Displayed"的行，将输出内容再作为输入能过管道转发给下一个grep
$ cat logcat.log | grep -n 'Displayed' | grep ms

# 匹配testgrep文件所有wirelessqa的行
$ grep -e wirelessqa testgrep

# '^wirelessqa'匹配testgrep文件所有以wirelessqa开头的行
$ grep -e ^[wirelessqa] testgrep

# 'wirelessqa$'匹配所有以wirelessqa结尾的行
$ grep -e [wirelessqa]$ testgrep

# .*一起用代表任意字符，'.*qa'匹配任何词以qa结尾的行
$ grep -e .*qa testgrep

# '\<wire'匹配包含以wire开头的单词的行
$ grep -e '\<wire' testgrep

# 'blog\>'匹配包含以blog结尾的单词的行
$ grep -e 'blog\>' testgrep

# 匹配aaa文件中'blog.csdn.net'的行的上下5行内容
$ grep -5 -e blog.csdn.net aaa
```

### chown、useradd、groupadd、userdel、usermod、passwd、groupdel（新建用户、用户组，给用户分配权限） 

```
# 添加新的账号
$ useradd 选项 用户名

# 删除账号
$ userdel 选项 用户名

# 修改帐号
$ usermod 选项 用户名

# 用户密码的管理
$ passwd 选项 用户名
Old password：******
New password：*******
Re-enter new password：*******

# 设置密码为空
$ passwd -d 用户名

# 添加新的组
$ groupadd 选项 用户组

# 删除组
$ groupdel 用户组

# 修改组
$ groupmod 选项 用户组

# 修改用户所在组（把某个用户改为新的用户组）
$ usermod -g 用户组 用户名

# 分配权限(修改某个目录的所属用户和用户组)
$ chown -R hadoop:hadoop /usr/hadoop/

# Linux中查看所有系统用户命令
cat /etc/passwd
```

### alias别名（很有用，便于记录常用命令）

```
# 查看当前设置了多少别名，直接输入alias
$ alias

# 添加别名，直接输入alias '别名'='命令'（注意等号之间不能有空格）
$ alias cdmanage='cd /Users/ben/workspace/manage/target'

# 删除一个别名
$ unalias cdmanage
```

```
# 永久保存alias的方式（本人是Mac操作系统）
# 进入用户根目录
$ cd /Users/ben
$ vi .bash_profile

# 编辑文件如下并保存
alias cdben="cd /Users/ben"
alias cdcore="cd /Users/ben/workspace/core"
alias cdmanage="cd /Users/ben/workspace/manage"
alias cdservice="cd /Users/ben/workspace/service"

alias cdmongo2="cd /Users/ben/dev/mongodb-osx-x86_64-2.6.9"
alias cdmongo3="cd /Users/ben/dev/mongodb-osx-x86_64-3.0.2"
alias cdredis="cd /Users/ben/dev/redis-3.0.0"
alias cdsearch01="cd /Users/ben/dev/search01"

# 使上面的别名立即生效
$ source .bash_profile
```

### Linux中修改IP地址
```
第一步：
# 修改/etc/sysconfig/network-scripts/ifcfg-eth0配置文件 
IPADDR=192.168.0.248 
GATEWAY=192.168.0.1 
DNS1=192.168.0.1 
第二步：
# 修改/etc/hosts文件 
192.168.0.248        hostname 
```

### Linux中修改主机名
```
第一步：
$ hostname oratest  
第二步：
# 修改/etc/sysconfig/network中的hostname  
第三步：
# 修改/etc/hosts文件 
记得重启!!!
```

### tar压缩/解压缩
```
# tar命令详解
-c: 建立压缩档案
-x：解压
-t：查看内容
-r：向压缩归档文件末尾追加文件
-u：更新原压缩包中的文件
# 这五个是独立的命令，压缩解压都要用到其中一个，可以和别的命令连用但只能用其中一个。

# 下面的参数是根据需要在压缩或解压档案时可选的。
-z：有gzip属性的
-j：有bz2属性的
-Z：有compress属性的
-v：显示所有过程
-O：将文件解开到标准输出
参数-f是必须的
-f: 使用档案名字，切记，这个参数是最后一个参数，后面只能接档案名。

# 压缩
# 这条命令是将所有.jpg的文件打成一个名为all.tar的包。-c是表示产生新的包，-f指定包的文件名。
$ tar -cf all.tar *.jpg
# 将目录里所有jpg文件打包成jpg.tar后，并且将其用gzip压缩，生成一个gzip压缩过的包，命名为jpg.tar.gz
$ tar –czf jpg.tar.gz *.jpg
# 这条命令是将所有.gif的文件增加到all.tar的包里面去。-r是表示增加文件的意思。 
$ tar -rf all.tar *.gif
# 这条命令是更新原来tar包all.tar中logo.gif文件，-u是表示更新文件的意思。
$ tar -uf all.tar logo.gif

# 查看
# 这条命令是在不解压的情况下查看all.tar包中所有文件，-t是列出文件的意思 
$ tar -tf all.tar

# 解压
# 这条命令是解出all.tar包中所有文件，-x是解开的意思
$ tar -xf all.tar
# 解压tar包
$ tar –xvf file.tar
# 解压tar.gz
$ tar -xzvf file.tar.gz
# 解压 tar.bz2
$ tar -xjvf file.tar.bz2
# 解压tar.Z
$ tar –xZvf file.tar.Z

# 解压到指定的目录，如果目录不存在就创建
# 将text.tar.gz 解压到 /home/app/test/ （绝对路径）下
$ tar -zxvf ./text.tar.gz -C /home/app/test/

# 解压rar
$ unrar e file.rar

# 压缩zip
$ zip aa.zip aa.sh

# 将某一个目录下的文件压缩成一个zip包
$ zip folder.zip folder/*

# 解压zip
$ unzip file.zip

# 解压zip到指定目录
# -n : 如果已有相同的文件存在，要求unzip命令不覆盖原先的文件
# -o : 如果已有相同的文件存在，要求unzip命令覆盖原先的文
$ unzip -n test.zip -d /tmp
$ unzip -o test.zip -d /tmp

# 总结 
1、*.tar 用 tar –xvf 解压 
2、*.gz 用 gzip -d或者gunzip 解压 
3、*.tar.gz和*.tgz 用 tar –xzf 解压 
4、*.bz2 用 bzip2 -d或者用bunzip2 解压 
5、*.tar.bz2用tar –xjf 解压 
6、*.Z 用 uncompress 解压 
7、*.tar.Z 用tar –xZf 解压 
8、*.rar 用 unrar e解压 
9、*.zip 用 unzip 解压

```

### 软链接
```
# 创建软链接
# ln -s a b 中的 a 就是源文件，b是链接文件名，其作用是当进入b目录，实际上是链接进入了a目录
# 值得注意的是执行命令的时候，应该是a目录已经建立，目录b没有建立。我最开始操作的是也把b目录给建立了，结果就不对了
# 如下面的示例，当我们执行命令cd /usr/local/java/的时候，实际上是进入了/software/java/
$ ln -s /software/java /usr/local/java

# 删除软链接（注意不是rm -rf b/）
rm -rf b
```

### 其他常用命令
```
# 创建文件夹命令
mkdir test

# 创建多级目录的命令
mkdir -p /software/temp/nginx

# 删除文件夹命令
rm -rf test 

# 复制文件夹命令
cp -rf /home/root/Downloads/* /home/root/tools
```

### MySQL数据库的备份与还原
```
# 备份
# root为用户名，user为要备份的数据库，自动会备份到/home/ben/文件目录下
$ mysqldump -u root -p user > /home/ben/user20160101.sql

# 压缩备份
$ mysqldump -u root -p user | gzip > user20160101.sql.gz 

# 还原
$ mysql -u root -p user < /home/ben/user20160101.sql 
```

参考文章：

- http://blog.csdn.net/column/details/linuxcommad.html?&page=2
- http://jingyan.baidu.com/article/d45ad148e801c769552b800c.html
- http://www.php-note.com/article/detail/831