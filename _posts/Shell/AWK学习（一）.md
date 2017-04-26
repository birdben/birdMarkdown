---
title: "AWK学习（一）"
date: 2016-08-15 00:15:18
tags: [AWK]
categories: [Shell]
---

Nginx/Apache/Lnmp网站常用日记统计命令
发布时间：April 13, 2012 // 分类：日记分析 // No Comments
Nginx需配置日记格式为Apache日志格式，便于分析。
1.访问次数最多的前10个IP。

```
awk '{print $1}' www.haiyun.me.log|sort|uniq -c|sort -rn|head -n 10
```

2.访问次数最多的10个页面。

```
awk '{print $7}' www.haiyun.me.log|sort|uniq -c|sort -rn|head -n 10
```

3.访问最多的时间，取前十个。

```
awk '{print $4}' www.haiyun.me.log|cut -c 14-18|sort|uniq -c|sort -rn|head -n10
```

4.查看下载次数最多的文件，显示前10个。

```
awk '{print $7}' www.haiyun.me.log|awk -F '/' '{print $NF}'|sort|uniq -c|sort -rn|head -n 10
#如统计请求链接去除awk -F '/' '{print $NF}'|sort|
```

5.统计网站流量，以M为单位。

```
awk '{sum+=$10} END {print sum/1024/1024}' www.haiyun.me.log
```

6.统计IP平均流量、总流量。

```
awk 'BEGIN {print"ip average total"}{a[$1]+=$10;b[$1]++}END{for(i in a)print i,a[i]/1024/1024/b[i]"MB",\
a[i]/1024/1024"MB"}' www.haiyun.me.log |column -t
```

7.用sed统计特定时间内日志，配合以上使用awk分析。

```
sed -n '/10\/Feb\/2012:18:[0-9][0-9]:[0-9][0-9]/,$p' www.haiyun.me.log
#截取二月10号18点后所有日志
sed -n '/10\/Feb\/2012:18:[0-9][0-9]:[0-9][0-9]/,/10\/Feb\/2012:20:[0-9][0-9]:[0-9][0-9]/p' \
www.haiyun.me.log
#截取二月10号18点到20点之间日志
```

8.统计404或403最多的网址。

```
awk '$9 ~ /403/ {print $7}' www.haiyun.me.log|sort|uniq -c|sort -rn|head -n 80
awk '$9 ~ /404/ {print $7}' www.haiyun.me.log|sort|uniq -c|sort -rn|head -n 80
```

参考文章：

- http://coolshell.cn/articles/9070.html
- http://www.cnblogs.com/kaituorensheng/p/3919212.html#_label1
