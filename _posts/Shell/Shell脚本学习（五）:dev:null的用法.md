---
title: "Shell脚本学习（五）/dev/null的用法"
date: 2016-08-17 22:47:11
tags: [Shell]
categories: [Shell]
---

在类似启动Tomcat的Shell脚本中我们经常会看到如：./startup.sh > /dev/null，正常这种用法都是执行./startup.sh启动脚本后，将log输出到控制台的。但是用./start.sh > /dev/null这种方式就类似daemon后台启动Tomcat的效果一样，但是启动的日志不会输出到log文件，也不会输出到控制台，而是输出到/dev/null。可以把/dev/null看作一个"黑洞"，它非常等价于一个只写文件，所有写入它的内容都会永远丢失，而尝试从它那儿读取内容则什么也读不到。所以我们是在任何地方都看不到这个Tomcat的启动log的。

```
1. 禁止标准输出
# 将catalina.out日志文件的内容输出到控制台
cat catalina.out
# 将catalina.out日志文件的内容输出到test.log文件中
cat catalina.out > test.log
# 将catalina.out日志文件的内容输出到/dev/null（黑洞）
cat catalina.out > /dev/null

2. 禁止输出ls文件列表
# 输出当前文件夹下的所有文件列表到ls.out文件中
ls > ls.out
# 将文件列表实际是输出到/dev/null（黑洞）
ls > /dev/null
```

这里在说明一下 < , > , >> , 2> 的区别

- 由 < 的右边读入参数档案
- 将原本由屏幕输出的正确数据输出到 > 右边的 file ( 文件名称 ) 或 device ( 装置，如 printer )去
- 将原本由屏幕输出的正确数据输出到 >> 右边，与 > 不同的是，该档案将不会被覆盖，而新的数据将以『增加的方式』增加到该档案的最后面
- 将原本应该由屏幕输出的错误数据输出到 2> 的右边去

```
# 正确的ls命令
$ ls -al
# 将显示的数据，不论正确或错误均输出到success.log（注意：2>&1中间不能有空格）
$ ls -al 1> success.log 2>&1
# 将显示的数据，正确的输出到success.log，错误的输出则予以丢弃，这里故意将ls命令的参数丢掉，让命令出错！
$ ls - 1> success.log 2> /dev/null
# 将显示的数据，正确的输出到success.log，错误的输出到error.log，这里故意将ls命令的参数丢掉，让命令出错！
$ ls - 1> success.log 2> error.log 
```