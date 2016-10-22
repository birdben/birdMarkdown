---
title: "Shell脚本学习（七）Shell中的特殊用法"
date: 2016-10-22 17:53:17
tags: [Shell]
categories: [Shell]
---

最近在网上看了别人写的Shell脚本，发现还是有很多语法看不懂需要百度才行，今天就总结一下我遇到的一些Shell特殊符号的用法问题

### Shell的特殊符号 $, $$, &, && 的用法


```
$$ : Shell本身的PID（ProcessID）
$! : Shell最后运行的后台Process的PID
$? : 最后运行的命令的结束代码（返回值）
$- : 使用Set命令设定的Flag一览
$* : Shell的所有参数列表
$@ : Shell的所有参数列表
$# : Shell的所有参数个数
$0 : Shell本身的文件名
$1～$n : Shell的各个参数值。$1是第1参数、$2是第2参数…
& : 放在启动参数后面表示设置此进程为后台进程
| : 管道 (pipeline) 连结上个指令的标准输出，做为下个指令的标准输入
&& : Shell命令之间使用 && 连接，实现逻辑与的功能
|| : Shell命令之间使用 || 连接，实现逻辑或的功能
1.命令之间使用 && 连接，实现逻辑与的功能。
2.如果左边的命令有返回值，该返回值保存在Shell变量 $? 中，只有在 && 左边的命令返回真（命令返回值 $? == 0），&& 右边的命令才会被执行。
3.只要有一个命令返回假（命令返回值 $? == 1），表示左边的命令执行失败，后面的命令就不会被执行。
下一条命令依赖前一条命令是否执行成功。如：在成功地执行一条命令之后再执行另一条命令，或者在一条命令执行失败后再执行另一条命令等。shell 提供了 && 和 || 来实现命令执行控制的功能，shell 将根据 && 或 || 前面命令的返回值来控制其后面命令的执行。
```

举例说明上述的Shell特殊符号的用法

```
#!/bin/bash

#
# AUTHOR: Yanpeng Lin
# DATE:   Mar 30 2014
# DESC:   lock a rotating file(static filename) and tail
#

PID=$( mktemp )
echo $PID
echo $(eval "cat $PID")
while true;
do
    CURRENT_TARGET=$( eval "echo $1" )
    echo $CURRENT_TARGET
    if [ -e ${CURRENT_TARGET} ]; then
        IO=`stat ${CURRENT_TARGET}`
        # 在后台运行监听{$CURRENT_TARGET}文件的变化，如果出错不输出错误信息，将最后执行的后台进程的ID输出到${PID}中（也就是tail -f {$CURRENT_TARGET} 2> /dev/null &这个命令的后台进程ID）
        tail -f {$CURRENT_TARGET} 2> /dev/null & echo $! > $PID;
        echo $!
    fi
	echo $PID
	echo $(eval "cat $PID")
    
    # as long as the file exists and the inode number did not change
    while [[ -e ${CURRENT_TARGET} ]] && [[ ${IO} = `stat -c %i ${CURRENT_TARGET}` ]]
    do
        CURRENT_TARGET=$( eval "echo $1" )
        #echo $CURRENT_TARGET
        sleep 0.5
    done
    # 如果kill命令执行失败，则输出错误信息，并且不会清空${PID}中的值
    if [ ! -z ${PID} ]; then
        kill `cat ${PID}` 2> /dev/null && echo > ${PID}
    fi
    sleep 0.5
done 2> /dev/null
rm -rf ${PID}
```