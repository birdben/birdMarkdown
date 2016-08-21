---
title: "Shell脚本学习（一）Shell命令基础"
date: 2016-08-08 23:55:59
tags: [Shell]
categories: [Shell]
---

```
echo $SHELL
# $SHELL是一个环境变量，它记录用户所使用的Shell类型。你可以用命令：

Shell-name
# 来转换到别的Shell，这里Shell-name是你想要尝试使用的Shell的名称，如ash等。这个命令为用户又启动了一个Shell，这个Shell在最初登录的那个Shell之后，称为下级的Shell或子Shell。

exit
# 可以退出这个子Shell。
```

先来个简单的例子吧，也就是我们程序猿最长说的helloworld

helloworld.sh

```
#!/bin/bash

# 注意："="号两边不能有空格，因为个人习惯问题，我就总喜欢在等号两边加上空格
demo="hello world"
# 在终端输出变量demo，也就是hello world
echo $demo
```

可以使用下面两种方式执行helloworld.sh，执行sh脚本之前需要检查权限问题，这里就不细说Linux权限问题了，否则执行会提示没有权限，具体请百度自行解决。

```
$ ./helloworld.sh
$ sh helloworld.sh
```

注意：这里使用不同的操作系统执行Shell脚本时，可能会遇到问题。拿我自己举例，我在Mac上能执行的Shell脚本可能在Ubuntu上无法执行，可能是因为Mac默认使用bash执行的，而Ubuntu默认使用的是sh执行的。简单一句话描述sh和bash的区别，sh是bash的“子集”，所以有的Shell脚本能用bash执行，而用sh执行就会报错。

解决办法

```
# 采用链接指向
$ ln -s /bin/bash /bin/sh
# 检查是否正确
$ ls -l /bin/sh
```

sh和bash的具体区别请参考：
http://www.cnblogs.com/hopeworld/archive/2011/03/29/1998488.html

#### 变量
Shell Script是一种弱类型语言，使用变量的时候无需首先声明其类型。新的变量会在本地数据区分配内存进行存储，这个变量归当前的Shell所有，任何子进程都不能访问本地变量。这些变量与环境变量不同，环境变量被存储在另一内存区，叫做用户环境区，这块内存中的变量可以被子进程访问。变量赋值的方式是：

variable\_name = variable\_value

如果对一个已经有值的变量赋值，新值将取代旧值。取值的时候要在变量名前加$，$variable_name可以在引号中使用，这一点和其他高级语言是明显不同的。如果出现混淆的情况，可以使用花括号来区分，例如：

echo "Hi, $demos"

就不会输出“Hi, hello worlds”，而是输出“Hi，”。这是因为Shell把$demos当成一个变量，而$demos未被赋值，其值为空。正确的方法是：

echo "Hi, ${demo}s"

单引号中的变量不会进行变量替换操作。
关于变量，还需要知道几个与其相关的Linux命令。

env用于显示用户环境区中的变量及其取值；set用于显示本地数据区和用户环境区中的变量及其取值；unset用于删除指定变量当前的取值，该值将被指定为NULL；export命令用于将本地数据区中的变量转移到用户环境区。


#### 数组
```
#!/bin/bash

# 一对括号表示是数组，数组元素用“空格”符号分割开。
a=(1 2 3 4 5)

###### 获取 ######
echo "获取"
a=(1 2 3 4 5)

# 用${#数组名[@或*]} 可以得到数组长度
echo ${#a[@]}
echo ${#a[*]}

# 用${数组名[下标]} 可以得到指定下标的值，下标是从0开始
echo ${a[2]}

# 用${数组名[@或*]} 可以得到整个数组内容
echo ${a[@]}
echo ${a[*]}

###### 赋值 ######
echo "赋值"
a=(1 2 3 4 5)

# 直接通过 数组名[下标] 就可以对其进行引用赋值
a[1]=100

# 如果下标不存在，自动添加新一个数组元素
a[1000]=1000

echo ${a[*]}
echo ${#a[*]}

###### 删除 ######
echo "删除"
a=(1 2 3 4 5)

# unset 数组[下标] 可以清除相应的元素
unset a[1]

echo ${a[*]}
echo ${#a[*]}

# unset 数组[下标] 不带下标，清除整个数据。
unset a

echo ${a[*]}
echo ${#a[*]}

###### 截取 ######
echo "截取"
a=(1 2 3 4 5)

# 截取数组 ${数组名[@或*]:起始位置:长度}，从下标0开始，截取长度为3，切片原先数组，返回是字符串，中间用“空格”分开
echo ${a[@]:0:3}
echo ${a[*]}

# 如果加上”()”，将得到切片数组，上面例子：c 就是一个新数据。
c=(${a[@]:1:4})
echo ${c[*]}
echo ${#c[*]}

###### 替换 ######
echo "替换"
a=(1 2 3 4 5)

# ${数组名[@或*]/查找字符/替换字符} 该操作不会改变原先数组内容，如果需要修改，可以看上面例子，重新定义数据。
echo ${a[@]/3/100}
echo ${a[@]}

# 如果需要需求，重新赋值给变量a
a=(${a[@]/3/100})
echo ${a[@]}

###### 根据分隔符拆分字符串为数组 ######
echo "根据分隔符拆分字符串为数组"
a="one,two,three,four"

# 要将$a按照","分割开，并且存入到新的数组中
OLD_IFS="$IFS"
IFS=","
arr=($a)
IFS="$OLD_IFS"
for s in ${arr[@]}
do
    echo "$s"
done

# arr=($a)用于将字符串$a分割到数组$arr ${arr[0]} ${arr[1]} ... 分别存储分割后的数组第1 2 ... 项 ，${arr[@]}存储整个数组。变量$IFS存储着分隔符，这里我们将其设为逗号 "," OLD_IFS用于备份默认的分隔符，使用完后将之恢复默认。
```

#### if语句

if语句格式

```
if …; then
	…
elif …; then
	…
else
	…
fi
```

与其他语言不同，Shell Script中if语句的条件部分要以分号来分隔。第三行中的[]表示条件测试，常用的条件测试有下面几种：

```
# 要注意条件测试部分中的空格。在方括号的两侧都有空格，在-f、-lt、=等符号两侧同样也有空格。如果没有这些空格，Shell解释脚本的时候就会出错。
[ -f "$file" ] : 判断$file是否是一个文件
[ -x "$file" ] : 判断$file是否存在且有可执行权限，同样-r测试文件可读性
[ -n "$a" ] : 判断变量$a是否有值
[ -z "$a" ] : 判断变量$a是否为空字符串
[ $a -lt 3 ] : 判断$a的值是否小于3，同样-gt和-le分别表示大于或小于等于
[ "$a" = "$b" ] : 判断$a和$b的取值是否相等
[ cond1 -a cond2 ] : 判断cond1和cond2是否同时成立，-o表示cond1和cond2有一成立
```

Shell逻辑运算符、逻辑表达式请参考：

- http://www.cnblogs.com/chengmo/archive/2010/10/01/1839942.html

if条件例子：

```
#!/bin/bash

# 打印终端命令行的所有参数
echo $*;

# 打印终端命令行的所有参数的个数
echo $#;

# 如果终端命令行的所有参数的个数小于3，就输出所有参数
if [ $# -lt 3 ]; then
	echo $*;
else
	echo $0;
	echo "参数过多不在控制台显示";
fi
```

在Shell中，脚本名称本身是$0，剩下的依次是$0、$1、$2…、${10}、${11}，等等。

- $* : 表示命令行的所有参数，不包括$0，也就是说不包括文件名的参数列表
- $# : 表示命令行参数的个数，不包括$0，其实也可以理解成是包括$0的索引下标

#### for，while，until循环语句

while循环语句格式

```
while [ cond1 ] && { || } [ cond2 ] …; do
	…
done
```

for循环语句格式

```
for var in …; do
	…
done

for (( cond1; cond2; cond3 )) do
	…
done
```

until循环语句格式

```
until [ cond1 ] && { || } [ cond2 ] …; do
	…
done
```

循环例子：

```
#!/bin/bash

###### while循环例子1 ######
echo "while循环例子1";

i=10;
while [[ $i -gt 5 ]]; do
    echo $i;
    ((i--));
done;

###### while循环例子2 ######
echo "while循环例子2";

# 循环读取/etc/hosts文件内容
while read line; do
    echo $line;
done < /etc/hosts;

###### for循环例子1 ######
echo "for循环例子1";
for((i=1;i<=10;i++)); do
    echo $i;
done;

###### for循环例子2 ######
echo "for循环例子2";

# seq 10 产生 1 2 3 。。。。10空格分隔字符串。
for i in $(seq 10); do
    echo $i;
done;

###### for循环例子3 ######
echo "for循环例子3";

# 根据终端输入的文件名来检查当前目录该文件是否存在
for file in $*; do
    if [ -f "$file" ]; then
        echo "INFO: $file exists"
    else
        echo "ERROR: $file not exists"
    fi
done;

###### until循环例子1 ######
echo "until循环例子1";

a=10;
until [[ $a -lt 0 ]]; do
    echo $a;
    ((a--));
done;
```

#### case语句

case/esac语句格式

```
case var in
    pattern 1 )
        … ;;
    pattern 2 )
        … ;;
    *)
        … ;;
esac
```

```
#!/bin/bash
case $1 in
    start | begin)
        echo "start something"  
    ;;
    stop | end)
        echo "stop something"  
    ;;
    *)
        echo "Ignorant"  
    ;;
esac
```

#### select交互语句

Bash提供了一种用于交互式应用的扩展select，用户可以从一组不同的值中进行选择。

select交互语句格式

```
select var in …; do
	break;
done
```

例如，下面这段程序的输出是：

```
#!/bin/bash

select ch in "begin" "end" "exit"; do
    case $ch in
        "begin")
            echo "start something"  
        ;;
        "end")
            echo "stop something"  
        ;;
        "exit")
            echo "exit"  
            break;
        ;;
        *)
            echo "Ignorant"  
        ;;
    esac
done;

## 注意这里交互输入要输入1，2，3，而不是beign，end，exit
# $ sh demo.sh
# 1) begin
# 2) end
# 3) exit
```

#### function语句

function语句格式

```
定义函数格式一：
functionname()
{
	…
}

定义函数格式二：
# 函数名前面多了个function关键字
function functionname() 
{
	…
}
```

函数使用例子：

```
#!/bin/bash

###### 函数定义 ######
echo "函数定义";

# 注意：所有函数在使用前必须定义。这意味着必须将函数放在脚本开始部分，直至shell解释器首次发现它时，才可以使用。调用函数仅使用其函数名即可。
function hello() {
    echo "Hello!";
}

function hello_param() {
    echo "Hello $1 !";
}
###### 函数调用 ######
# 函数调用
echo "函数调用";
hello;

###### 参数传递 ######
echo "函数传参调用";
hello_param ben;

###### 函数文件 ######
echo "函数文件调用";
# 调用函数文件，点和demo_call之间有个空格
. demo_call.sh;
# 调用函数
callFunction ben;

###### 载入和删除 ######
echo "载入和删除";

# 用unset functionname 取消载入
# unset callFunction;
# 因为已经取消载入，所以会出错
# callFunction ben;

###### 参数读取 ######
echo "参数读取";

# 参数读取的方式和终端读取参数的方式一样
funWithParam(){
    echo "The value of the first parameter is $1 !"
    echo "The value of the second parameter is $2 !"
    echo "The value of the tenth parameter is $10 !"
    echo "The value of the tenth parameter is ${10} !"
    echo "The value of the eleventh parameter is ${11} !"
    echo "The amount of the parameters is $# !"
    echo "The string of the parameters is $* !"
}
funWithParam 1 2 3 4 5 6 7 8 9 34 73

###### 函数return ######
echo "函数return";

funWithReturn(){
    echo "The function is to get the sum of two numbers..."
    echo -n "Input first number: "
    read aNum
    echo -n "Input another number: "
    read anotherNum
    echo "The two numbers are $aNum and $anotherNum !"
    return $(($aNum+$anotherNum))
}
funWithReturn
# 函数返回值在调用该函数后通过 $? 来获得
echo "The sum of two numbers is $? !"
```

函数文件demo_call.sh

```
#!/bin/bash
function callFunction() {
    echo "callFunction $1 !";
    return 1;
}
```

参考文章：

- http://www.cnblogs.com/suyang/archive/2008/05/18/1201990.html
- http://www.cnblogs.com/chengmo/archive/2010/09/30/1839632.html
- http://www.cnblogs.com/FlyFive/p/3640293.html
