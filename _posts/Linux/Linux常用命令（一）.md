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

# 找到被删除的却还被进程占用的文件
$ lsof | grep deleted
```

### ps查看进程
```
# 查看Linux端口号
# ps 命令用于查看当前正在运行的进程
# grep 是搜索
$ ps -ef | grep 80

# 根据进程ID查看进程
$ pe -eLf | grep PID

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

### nohup后台启动
```
# 格式：nohup command &
# 后台启动Kibana，会在执行此命令的目录下生成一个nohup.out日志文件
$ nohup kibana &

# 后台启动Logstash，并且指定日志输出到logstash.out文件中
$ nohup logstash -f ${LOGSTASH_HOME}/conf/logstash.conf 2>${LOGSTASH_HOME}/bin/logstash.out 1>${LOGSTASH_HOME}/bin/logstash.out &
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
00:36:48.658266 IP (tos 0x0, ttl 64, id 15221, offset 0, flags [DF], proto TCP (6), length 40)
    10.211.55.4.32903 > 61.135.169.125.443: Flags [.], cksum 0xe7a0 (correct), ack 4785, win 343, length 0

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

# 参数描述：
! / -not: 非处理
-maxdepth : 搜索层级
-type : 搜索文件的类型，f：文件，l：链接文件，d：文件夹
-name : 文件名，可以使用通配符
-iname : 文件名，忽略文件名大小写
-mtime : 文件的最后modify时间
-atime : 文件的最后access时间
-ctime : 文件的最后create时间
-path : （排除）搜索的文件路径
-prune -o : 配合-path一起使用，排除搜索的路径
-size : 文件的大小

# 搜索当前目录中，所有文件名以my开头的文件 -name（文件名需要带后缀）
$ find . -name 'my*'

# 搜索当前目录中，所有文件名以my开头的文件，并显示它们的详细信息。
$ find . -name 'my*' -ls

# 以文件大小来查找 -size n
$ find . -size +1000000c（在当前目录下查找文件长度大于1 M字节的文件 ）

# 在当前目录及所有子目录中查找filename(忽略大小写)
$ find . -iname "Hessian.properties"

# -prune -o用法
# find condition1 -prune -o (condition2 -prune -o condition3 -prune -o) condition4 -print
# 查找当前目录下的properties文件，排除yunyu-stt文件夹
$ find . -path "./yunyu-stt" -prune -o -name "*.properties" -print

# 查找当前目录下的properties文件，排除yunyu-stt和yunyu-task文件夹
$ find . -path "./yunyu-stt" -prune -o -path "./yunyu-task" -prune -o -name "*.properties" -print

# 查找当前目录下的properties文件，排除yunyu-stt和yunyu-task文件夹下的test文件夹
$ find . -path "./yunyu-stt/*/test" -prune -o -path "./yunyu-task/*/test" -prune -o -name "*.properties" -print

# 按照多种条件来查找
# type是file，但不是link
# maxdepth是1
# name是*，但不是gz，zip，pid结尾的文件
# -mtime是表示文件修改时间为大于1天的文件，即距离当前时间2天（48小时）之外的文件
$ find . -maxdepth 1 -type f -name "*" ! -name "*.gz"  ! -name "*.zip" ! -name "*.pid" ! -type l -mtime +1
```

### find结合xargs命令批量操作文件

```
# 删除当前目录下除了test.txt的其他所有文件
$ find . -type f -not -name test.txt | xargs rm -v

# 默认情况下，find命令每输出一个文件名，后面都会接着输出一个换行符 ('n')，因此我们看到的find的输出都是一行一行的。xargs默认是以空白字符（空格，TAB，换行符）来分割记录的，如果文件名中带有空格这样会被xargs分割开，删除的时候就会找不到该文件了。所以让find在打印出一个文件名之后接着输出一个NULL字符 ('') 而不是换行符，然后再告诉xargs也用NULL字符来作为记录的分隔符。这样就解决了上面的问题。也就是find的-print0和xargs的-0的作用。
$ find . -type f -not -name test.txt -print0 | xargs -0 rm -v

# 查找当前文件下log文件，并且批量替换文件内容martini为middleware
$ find . -name '*.log' | xargs sed -i 's/martini/middleware/g'

# 查找当前目录下的properties文件，排除yunyu-stt和yunyu-task文件夹，并且搜索properties文件中包含127.0.0.1:2181
$ find . -path "./yunyu-stt" -prune -o -path "./yunyu-task" -prune -o -name "*.properties" -print | xargs grep -e "127.0.0.1:2181"

# 查找当前目录下的properties文件，排除yunyu-stt和yunyu-task文件夹下的test文件夹，并且搜索properties文件中包含127.0.0.1:2181
$ find . -path "./yunyu-stt/*/test" -prune -o -path "./yunyu-task/*/test" -prune -o -name "*.properties" -print | xargs grep -e "127.0.0.1:2181"
```

### grep文件内容查找相关

```
$ grep [options]

[options]主要参数：
－c：只输出匹配行的计数。
－e <范本样式>：指定字符串作为查找文件内容的范本样式。
－I：不区分大小写(只适用于单字符)。
－h：查询多文件时不显示文件名。
－l：查询多文件时只输出包含匹配字符的文件名。
－n：显示匹配行及行号。
－s：不显示不存在或无匹配文本的错误信息。
－v：显示不包含匹配文本的所有行。
－o：只输出文件中匹配到的部分。

-a 不要忽略二进制数据。
-A<显示列数> 除了显示符合范本样式的那一行之外，并显示该行之后的内容。
-b 在显示符合范本样式的那一行之外，并显示该行之前的内容。
-c 计算符合范本样式的列数。
-C<显示列数>或-<显示列数> 除了显示符合范本样式的那一列之外，并显示该列之前后的内容。
-d<进行动作> 当指定要查找的是目录而非文件时，必须使用这项参数，否则grep命令将回报信息并停止动作。
-e<范本样式> 指定字符串作为查找文件内容的范本样式。
-E 将范本样式为延伸的普通表示法来使用，意味着使用能使用扩展正则表达式。
-f<范本文件> 指定范本文件，其内容有一个或多个范本样式，让grep查找符合范本条件的文件内容，格式为每一列的范本样式。
-F 将范本样式视为固定字符串的列表。
-G 将范本样式视为普通的表示法来使用。
-h 在显示符合范本样式的那一列之前，不标示该列所属的文件名称。
-H 在显示符合范本样式的那一列之前，标示该列的文件名称。
-i 忽略字符大小写的差别。
-l 列出文件内容符合指定的范本样式的文件名称。
-L 列出文件内容不符合指定的范本样式的文件名称。
-n 在显示符合范本样式的那一列之前，标示出该列的编号。
-q 不显示任何信息。
-R/-r 此参数的效果和指定“-d recurse”参数相同。
-s 不显示错误信息。
-v 反转查找。
-w 只显示全字符合的列。
-x 只显示全列符合的列。
-y 此参数效果跟“-i”相同。
-o 只输出文件中匹配到的部分。
--exclude-dir 搜索文件内容时，排除指定的文件夹

grep正则表达式元字符集：

- ^ 锚定行的开始 如：'^grep'匹配所有以grep开头的行。 
- $ 锚定行的结束 如：'grep$'匹配所有以grep结尾的行。 
- . 匹配一个非换行符的字符 如：'gr.p'匹配gr后接一个任意字符，然后是p。 
- * 匹配零个或多个先前字符 如：'*grep'匹配所有一个或多个空格后紧跟grep的行。 .*一起用代表任意字符。
- [] 匹配一个指定范围内的字符，如'[Gg]rep'匹配Grep和grep。 
- [^] 匹配一个不在指定范围内的字符，如：'[^A-FH-Z]rep'匹配不包含A-R和T-Z的一个字母开头，紧跟rep的行。 
- \(..\) 标记匹配字符，如'\(love\)'，love被标记为1。 
- \ 锚定单词的开始，如:'\匹配包含以grep开头的单词的行。 
- \> 锚定单词的结束，如'grep\>'匹配包含以grep结尾的单词的行。 
- x\{m\} 重复字符x，m次，如：'0\{5\}'匹配包含5个o的行。 
- x\{m,\} 重复字符x,至少m次，如：'o\{5,\}'匹配至少有5个o的行。 
- x\{m,n\}重复字符x，至少m次，不多于n次，如：'o\{5,10\}'匹配5--10个o的行。
- \w 匹配文字和数字字符，也就是[A-Za-z0-9]，如：'G\w*p'匹配以G后跟零个或多个文字或数字字符，然后是p。
- \b 单词锁定符，如: '\bgrep\b'只匹配grep。

关于匹配的实例：
# 统计所有以“48”字符开头的行有多少
$ grep -c "48" test.txt

# 不区分大小写查找“May”所有的行）
$ grep -i "May" test.txt

# 显示行号；显示匹配字符“48”的行及行号，相同于 nl test.txt |grep 48）
$ grep -n "48" test.txt

# 显示输出没有字符“48”所有的行）
$ grep -v "48" test.txt

# 显示输出字符“471”所在的行）
$ grep "471" test.txt

# 显示输出以字符“48”开头，并在字符“48”后是一个tab键所在的行
$ grep "48;" test.txt

# 显示输出以字符“48”开头，第三个字符是“3”或是“4”的所有的行）
$ grep "48[34]" test.txt

# 显示输出行首不是字符“48”的行）
$ grep "^[^48]" test.txt

# 设置大小写查找：显示输出第一个字符以“M”或“m”开头，以字符“ay”结束的行）
$ grep "[Mm]ay" test.txt

# 显示输出第一个字符是“K”，第二、三、四是任意字符，第五个字符是“D”所在的行）
$ grep "K…D" test.txt

# 显示输出第一个字符的范围是“A-D”，第二个字符是“9”，第三个字符的是“D”的所有的行
$ grep "[A-Z][9]D" test.txt

# 显示第一个字符是3或5，第二三个字符是任意，以1998结尾的所有行
$ grep "[35]..1998" test.txt

# 模式出现几率查找：显示输出字符“4”至少重复出现两次的所有行
$ grep "4\{2,\}" test.txt

# 模式出现几率查找：显示输出字符“9”至少重复出现三次的所有行
$ grep "9\{3,\}" test.txt

# 模式出现几率查找：显示输出字符“9”重复出现的次数在一定范围内，重复出现2次或3次所有行
$ grep "9\{2,3\}" test.txt

# 显示输出空行的行号
$ grep -n "^$" test.txt

# 如果要查询目录列表中的目录 同：ls -d *
$ ls -l |grep "^d"

# 在一个目录中查询不包含目录的所有文件
$ ls -l |grep "^d[d]"

# 查询其他用户和用户组成员有可执行权限的目录集合
$ ls -l |grep "^d…..x..x"

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

# 显示/Users/ben目录下的文件(不含子目录)包含blog的行
$ grep blog /Users/ben

# 显示/Users/ben目录下的文件(包含子目录)包含blog的行
$ grep -r blog /Users/ben

# 结合sort和uniq一起使用可以排序并去重
# 这里使用了正则，"."是匹配一个非换行符的字符
$ cat adlog_kafka_20161204.log | grep -o 'logs.....' | sort | uniq

# 统计各行在文件中出现的次数
$ sort file.txt | uniq -c

# 在文件中找出重复的行
$ sort file.txt | uniq -d

# 搜索当前文件夹下的匹配127.0.0.1:2181的内容，排除yunyu-stt文件夹下的文件
$ grep -r "127.0.0.1:2181" . --exclude-dir="./yunyu-stt"


# 或操作
# 找出文件（filename）中包含123或者包含abc的行
$ grep -E '123|abc' filename

# 用egrep同样可以实现
$ egrep '123|abc' filename

# awk 的实现方式
$ awk '/123|abc/' filename

# 与操作
# 显示既匹配 pattern1 又匹配 pattern2 的行
$ grep pattern1 files | grep pattern2

# 其他操作
# 不区分大小写地搜索。默认情况区分大小写
$ grep -i pattern files

# 只列出匹配的文件名
$ grep -l pattern files

# 列出不匹配的文件名
$ grep -L pattern files

# 只匹配整个单词，而不是字符串的一部分（如匹配‘magic’，而不是‘magical’）
$ grep -w pattern files

# 匹配的上下文分别显示[number]行
$ grep -C number pattern files
```

### sort排序用法

```
# sort参数
-f : 忽略大小写的差异，例如 A 与 a 视为编码相同
-b : 忽略最前面的空格符部分
-M : 以月份的名字来排序，例如 JAN, DEC 等等的排序方法
-n : 使用『纯数字』进行排序(默认是以文字型态来排序的)
-r : 反向排序
-u : 就是 uniq ，相同的数据中，仅出现一行代表
-t : 分隔符，默认是用 [tab] 键来分隔
-k : 以那个区间 (field) 来进行排序的意思

# 在test.txt文本中匹配不是shenzhen的行，获取第3列和第4列的内容，然后按照第4列进行排序
$ cat test.txt | grep -v shenzhen | awk '{print $4, $3}' | sort
20 shanghai
28 beijing
28 beijing
30 shanghai

# 在test.txt文本中匹配不是shenzhen的行，获取第3列和第4列的内容，这里指定输出后的第2列进行排序，即awk的$3
$ cat test.txt | grep -v shenzhen | awk '{print $4, $3}' | sort -k 2
28 beijing
28 beijing
20 shanghai
30 shanghai
```

### uniq去重用法

```
# uniq参数
-i : 忽略大小写字符的不同
-c : 列出重复出现的次数
-u : 只显示不重复行
-d : 仅显示重复行

# 去重后的结果
$ cat test.txt | grep -v shenzhen | awk '{print $4, $3}' | sort | uniq
20 shanghai
28 beijing
30 shanghai

# 列出重复出现的次数
$ cat test.txt | grep -v shenzhen | awk '{print $4, $3}' | sort | uniq -c
1 20 shanghai
2 28 beijing
1 30 shanghai

# 只显示不重复行
$ cat test.txt | grep -v shenzhen | awk '{print $4, $3}' | sort | uniq -u
20 shanghai
30 shanghai

# 仅显示重复行
$ cat test.txt | grep -v shenzhen | awk '{print $4, $3}' | sort | uniq -d
28 beijing
```

### sed在线编辑器

```
sed [-nefr] [动作]

# 选项与参数：
-n : 使用安静(silent)模式。在一般 sed 的用法中，所有来自 STDIN 的数据一般都会被列出到终端上。但如果加上 -n 参数后，则只有经过sed 特殊处理的那一行(或者动作)才会被列出来。
-e : 直接在命令列模式上进行 sed 的动作编辑；
-f : 直接将 sed 的动作写在一个文件内， -f filename 则可以运行 filename 内的 sed 动作；
-r : sed 的动作支持的是延伸型正规表示法的语法。(默认是基础正规表示法语法)
-i : 直接修改读取的文件内容，而不是输出到终端。

# 动作说明：[n1[,n2]]function
# n1, n2 : 不见得会存在，一般代表『选择进行动作的行数』，举例来说，如果我的动作是需要在 10 到 20 行之间进行的，则『 10,20[动作行为] 』

# function：
a : 新增， a 的后面可以接字串，而这些字串会在新的一行出现(目前的下一行)～
c : 取代， c 的后面可以接字串，这些字串可以取代 n1,n2 之间的行！
d : 删除，因为是删除啊，所以 d 后面通常不接任何咚咚；
i : 插入， i 的后面可以接字串，而这些字串会在新的一行出现(目前的上一行)；
p : 列印，亦即将某个选择的数据印出。通常 p 会与参数 sed -n 一起运行～
s : 取代，可以直接进行取代的工作哩！通常这个 s 的动作可以搭配正规表示法！例如 1,20s/old/new/g 就是啦！


# 列出test.txt文件的内容，并且显示将2-5行的结果
$ nl ~/Downloads/test.txt | sed -n '2,5p'

# 列出test.txt文件的内容，并且显示将2-5行删除后的结果
$ nl ~/Downloads/test.txt | sed '2,5d'

# 列出test.txt文件的内容，并且显示在第2行之前插入新的记录后的结果
$ nl ~/Downloads/test.txt | sed '2i 111'

# 搜索test.txt文件内容匹配beijing的，并且显示匹配的结果
$ nl ~/Downloads/test.txt | sed -n '/beijing/p'

# 搜索test.txt文件内容匹配beijing的，并且显示替换匹配的结果
$ nl ~/Downloads/test.txt | sed -n '/beijing/{s/beijing/shanghai/;p}'

# sed 's/要被取代的字串/新的字串/g'

# 搜索test.txt文件内容匹配beijing的，并且显示替换匹配的结果
$ nl ~/Downloads/test.txt | sed -n 's/beijing/shanghai/gp'

# 替换源文件的内容
$ sed -i 's/beijing/shanghai/g' ~/Downloads/test.txt

# 替换Downloads目录下所有文件，匹配内容beijing，替换为shanghai
$ sed -i 's/beijing/shanghai/g' `grep -rn "beijing" ~/Downloads/`
```

### sed结合grep命令批量替换文件内容

```
# 将当前目录(包括子目录)中所有以test开头的txt文件中的zzzz字符串替换为oooo字符串
sed -i 's/zzzz/oooo/g' `grep zzzz -rl --include="test_*.txt" ./`

# sed参数
-i : 表示操作的是文件，``括起来的grep命令，表示将grep命令的的结果作为操作文件
s/zzzz/oooo/ : 表示查找zzzz并替换为oooo
g : 表示一行中有多个zzzz的时候，都替换，而不是仅替换第一个

# grep参数
-r : 表示查找所有子目录
-l : 表示仅列出符合条件的文件名，用来传给sed命令做操作
--include="test_*.txt" : 表示仅查找test开头的txt文件
./ : 表示要查找的根目录为当前目录
```

### du定位大文件

```
# 查看当前目录下的所有文件大小（不包括下级目录，x参数会去除掉mount上去的目录）
# 如果有子文件夹，那就进去继续执行'du -shx *'一级一级地找
$ du -shx *

# 如果有文件被删除，却被某进程占用，并且还在写
# 找到被删除的却还被进程占用的文件，进程也被列出，把相关的进程重启一遍，空间就被释放了。
$ lsof | grep deleted

# 参数
# -b : 以Byte单位输出
# -k : 以KB为单位输出
# -m : 以MB为单位输出
# -h : 以K，M，G为单位显示
# -s : 仅显示总计，只列出最后加总的值

# 找出大于2000m的文件夹
$ du -m ~/Downloads | awk '$1 > 2000'
3048	/Users/yunyu/Downloads/develop
7966	/Users/yunyu/Downloads/install
17671	/Users/yunyu/Downloads/yunyu
31673	/Users/yunyu/Downloads

# 以可阅读的单位显示某文件夹的大小
$ du -sh ~/Downloads
31G		/Users/yunyu/Downloads

# 以可阅读的单位显示某文件的大小
$ du -h command.log
52K		command.log

# 统计多个文件的大小总和
$ du -ch command.log command_1.log
52K		command.log
4.0K	command_1.log
56K		total

# 找出大于2000m的文件夹，并且按照文件大小排序
du -m ~/Downloads | awk '$1 > 2000' | sort -nr | more
31673   /Users/yunyu/Downloads
17671   /Users/yunyu/Downloads/yunyu
7966    /Users/yunyu/Downloads/install
3048    /Users/yunyu/Downloads/develop
```

### split拆分合并文件

```
# 将大文件bigfile.tar按照每100m拆分成多个小文件
# 拆分后的小文件命名类似：split-bigfile.aa, split-bigfile.ab, split-bigfile.ac等等
split -b 100m bigfile.tar split-bigfile.
# 将多个小文件split-bigfile.*合并成一个大文件bigfile.tar
cat split-bigfile.* > bigfile.tar
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

# 修改用户所在组（把某个用户改为新的用户组，这样会离开其他用户组）
$ usermod -g 用户组 用户名

# 修改用户所在组（-a代表append，给某个用户添加一个新的组，而不必离开 其他用户组）
$ usermod -a -G 用户组 用户名

# 分配权限(修改某个目录的所属用户和用户组)
$ chown -R hadoop:hadoop /usr/hadoop/

# Linux中查看所有系统用户命令
$ cat /etc/passwd

# Linux中查看所有组用户
$ cat /etc/group

# 查看当前用户所属组
$ groups

# 查看root用户所属组
$ groups root

# 查看当前用户
$ whoami

# 添加用户到用户组
$ gpasswd -a 用户名 用户组

# 从用户组中删除用户
$ gpasswd -d 用户名 用户组
```

### chmod

方式一：用包含字母和操作符表达式的文字设定法

```
其语法格式为：chmod [who] [opt] [mode] 文件/目录名 

# who表示对象，是以下字母中的一个或组合：
u：表示文件所有者 
g：表示同组用户
o：表示其它用户
a：表示所有用户

# opt则是代表操作：
+：添加某个权限
-：取消某个权限
=：赋予给定的权限，并取消原有的权限

# mode则代表权限：
r：可读
w：可写
x：可执行  表示只有当该档案是个子目录或者该档案已经被设定过为可执行

# 为同组用户增加对文件test.txt的读写权限
$ chmod g+rw test.txt

# 给test.sh执行权限
$ chmod +x test.sh
```

方式二：用数字设定法

```
其语法格式为：chmod [mode] 文件名 或者 chmod UPO 文件名

UPO分别表示User、Group、及Other的权限

关键是mode的取值，其实很简单，我们将rwx看成二进制数，如果有则有1表示，没有则有0表示。
那么rwx r-x r-- 则可以表示成为：111 101 100 再将其每三位转换成为一个十进制数，就是754。

可读 : w=4
可写 : r=2 
可执行 : x=1

例如，我们想让test.txt这个文件的权限为： 
        自己  同组用户   其他用户 
可读     是     是         是 
可写     是     是            
可执行

那么，我们先根据上表得到权限串为：rw-rw-r--，那么转换成二进制数就是110 110 100，
再每三位转换成为一个十进制数，就得到664。
因此我们执行命令：chmod 664 a.txt

例如：
-rw------- (600) -- 只有属主有读写权限
-rw-r--r-- (644) -- 只有属主有读写权限；而属组用户和其他用户只有读权限
-rwx------ (700) -- 只有属主有读、写、执行权限
-rwxrwx--- (770) -- 只有属主和属组用户有读、写、执行权限
-rwxrwxrwx (777) -- 所有用户都有读、写、执行权限

# 给test.sh所有用户读、写、执行权限
$ chmod 777 test.sh
```

### 查看用户登录信息

```
$ w
$ w username

18:40  up 11 days, 21:35, 5 users, load averages: 3.35 2.94 2.92
USER     TTY      FROM              LOGIN@  IDLE WHAT
ben      console  -                16 816  11days -
ben      s000     -                16 816      - w
ben      s001     -                14:10      12 -bash
ben      s003     -                21 816     13 -bash
ben      s004     -                16:28       2 -bash

# 输出内容
USER :登陆的用户名；
TTY :登陆终端；
FROM :从哪个IP地址登录；
LOGIN@ :登陆时间；
IDLE :用户闲置时间；
JCPU :指的是和该终端连接的所有进程占用的时间。这个时间里并不包括过去的后台作业时间，但却包括当前正在运行的后台作业所占用的时间；
PCPU :是指当前进程所占用的时间；
WHAT :当前正在运行的命令；

$ who

ben      console  Aug 16 21:05
ben      ttys000  Aug 16 23:26
ben      ttys001  Aug 28 14:10
ben      ttys003  Aug 21 13:25
ben      ttys004  Aug 28 16:28

# 输出内容
- 用户名
- 登录终端
- 登录时间（登录来源IP地址）

# 查看当前登录和过去登录的用户信息
# 注释：last命令默认读取/var/log/wtmp文件数据
$ last

ben       ttys005                   Sun Aug 28 17:34 - 17:53  (00:19)
ben       ttys005                   Sun Aug 28 17:21 - 17:29  (00:07)
ben       ttys004                   Sun Aug 28 16:28   still logged in
ben       ttys001                   Sun Aug 28 14:10   still logged in
ben       ttys002                   Wed Aug 24 23:54 - 18:25 (3+18:31)

# 输出内容
- 用户名
- 登录终端
- 登录IP
- 登录时间
- 退出时间（在线时间）

# 查看所有用户最后一次登录时间（Ubuntu好用，Mac不好用，Mac下没有/var/log/lastlog文件）
# 注释：lastlog命令默认读取/var/log/lastlog文件内容
$ lastlog

# 输出内容
- 用户名
- 登录终端
- 登录IP
- 最后一次登录时间
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
- http://www.cnblogs.com/xiaouisme/archive/2012/11/09/2762543.html
- http://blog.csdn.net/klyz1314/article/details/20250487