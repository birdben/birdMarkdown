---
title: "AWK学习（二）"
date: 2017-03-11 14:09:05
tags: [AWK]
categories: [Shell]
---

### awk用法

注意：使用awk标准版可以不必安装gawk，使用gawk扩展功能必须要先安装gawk

```
# Ubuntu环境
$ sudo apt-get install gawk

# Mac环境
$ brew install gawk
```

### awk命令行格式

```
# 方式一：awk命令直接指定过滤规则
awk [options] file ...

# 方式二：指定awk的脚本文件，脚本文件内是指定的过滤规则
awk [options] -f file ....
```

方式一：awk命令直接指定过滤规则

```
awk 'BEGIN print{"HEAD1\tHEAD2\tHEAD3\tHEAD4\n"} {print} END print{"END1\tEND2\tEND3\tEND4\n"}' test.txt

```

方式二：指定awk的脚本文件，脚本文件内是指定的过滤规则

```
awk -f command.awk marks.txt
```

command.awk

```
BEGIN print{"HEAD1\tHEAD2\tHEAD3\tHEAD4\n"} {print} END print{"END1\tEND2\tEND3\tEND4\n"} test.txt
```

### awk结构

一个awk程序包含一系列的  模式 {动作指令} 或是函数定义。

```
# 动作指令需要以{}引起来
$ awk 'BEGIN {print "start"} {print} END {print "end"}' test.txt

# BEGIN rule(s)
BEGIN
{
	print "start"
}

# Rule(s)
{
	# $0是隐含参数，输出整行内容
	print $0
}

# END rule(s)
END
{
	print "end"
}
```

### awk原理

1). awk逐行扫描文件，从第一行到最后一行，寻找匹配特定模式的行，并在这些行上进行你想要的操作。
2). awk基本结构包括模式匹配(用于找到要处理的行)和处理过程(即处理动作)。
       pattern  {action}
# 提示：awk读取文件内容的每一行时，将对比改行是否与给定的模式相匹配，如果匹配则执行处理过程，否则对该行不做任何处理。
如果没有指定处理脚本，则把匹配的行显示到标准输出，即默认处理动作是print打印行；
如果没有指定模式匹配，则默认匹配所有数据。
3). awk有两个特殊的模式：BEGIN和END，他们被放置在没有读取任何数据之前以及在所有数据读取完成以后执行。

### 标准awk选项

```
# -v : 该选项将一个值赋予一个变量，它会在程序开始之前进行赋值，可以通过--dump-variables[=file]输出出来
$ awk -v bird=birdben 'BEGIN {print "bird=" bird}'
bird=birdben
```

### 标准awk内置变量

```
# ARGC : awk命令行参数个数
$ awk 'BEGIN {print "ARGC=" ARGC}'
ARGC=1

$ awk 'BEGIN {print "ARGC=" ARGC}' test1 test2
ARGC=3

# ARGV : 命令行参数数组，存储命令行参数的数组，索引范围从0 - ARGC - 1。
$ awk 'BEGIN {print "ARGV[0]=" ARGV[0]}'
ARGV[0]=awk

$ awk 'BEGIN {print "ARGV[1]=" ARGV[1] "\t" "ARGV[2]=" ARGV[2]}' test1 test2
ARGV[1]=test1	ARGV[2]=test2

# 循环输出ARGV数组中的参数值
$ awk 'BEGIN {
 	for (i = 0; i <= ARGC - 1; i++) { 
    	print "ARGV[" i "] = " ARGV[i] 
    	printf "ARGV[%d] = %s\n", i, ARGV[i] 
	} 
}' test1 test2
ARGV[0] = awk
ARGV[0] = awk
ARGV[1] = test1
ARGV[1] = test1
ARGV[2] = test2
ARGV[2] = test2

# 这里顺便说一下print和printf函数的区别
# print函数是不格式化直接输出函数，默认自动换行
# printf()函数是格式化输出函数，默认不会自动换行

# 上面是printf()函数的简写方式，完整的写法应该如下
$ awk 'BEGIN {
	for (i = 0; i <= ARGC - 1; i++) { 
		print "ARGV[" i "] = " ARGV[i]
		printf("ARGV[%d] = %s\n", i, ARGV[i]) 
	} 
}' test1 test2
ARGV[0] = awk
ARGV[0] = awk
ARGV[1] = test1
ARGV[1] = test1
ARGV[2] = test2
ARGV[2] = test2

# CONVFMT : 此变量表示数据转换为字符串的格式，其默认值为 %.6g
$ awk 'BEGIN { print "Conversion Format =" CONVFMT }'
Conversion Format = %.6g

# ENVIRON : 此变量是与环境变量相关的关联数组变量，以key-value的方式查看系统环境变量的值。
$ awk 'BEGIN { print ENVIRON["JAVA_HOME"] }'
/Library/Java/JavaVirtualMachines/jdk1.8.0_121.jdk/Contents/Home

# FILENAME : 此变量表示当前文件名称。
# 注意：这里一定要是END才能读取到文件名，因为在BEGIN开始快还没有开始读取文件test.txt的内容，也就是FILENAME是未定义的。
$ awk 'END {print "FILENAME = " FILENAME}' test.txt
FILENAME = test.txt

# FS : 此变量表示输入的数据域之间的分隔符，其默认值是空格。你可以使用 -F 命令行选项改变它的默认值。
$ awk 'BEGIN {print "FS = " FS}' | cat -vte
FS =  $

# NF : 此变量表示当前输入记录中域的数量。（简单理解，域：当前行用分隔符分开数据的就是数据域，如下面的例子，One，Two，Three都是数据域）
# 输出每一行的数据域数量
$ echo -e "One Two\nOne Two Three\nOne Two Three Four" | awk '{print "NF = " NF}'
NF = 2
NF = 3
NF = 4

# 输出每一行的数据域数量大于2的
$ echo -e "One Two\nOne Two Three\nOne Two Three Four" | awk 'NF > 2'
$ echo -e "One Two\nOne Two Three\nOne Two Three Four" | awk 'NF > 2 {print}'
One Two Three
One Two Three Four

# NR : 此变量表示当前记录的数量。
# 输出每一行的当前记录数量，也就是当前行的游标
$ echo -e "One Two\nOne Two Three\nOne Two Three Four" | awk '{print "NR = " NR}'
NR = 1
NR = 2
NR = 3

# 输出每一行的当前记录的游标大于2的
$ echo -e "One Two\nOne Two Three\nOne Two Three Four" | awk 'NR > 2 {print}'
One Two Three Four

# FNR : 该变量与 NR 类似，不过它是相对于当前文件而言的。此变量在处理多个文件输入时有重要的作用。每当从新的文件中读入时 FNR 都会被重新设置为 0。
$ awk '{print "FNR = " FNR "\t" "NR = " NR}' test.txt test1.txt
FNR = 1	NR = 1
FNR = 2	NR = 2
FNR = 3	NR = 3
FNR = 4	NR = 4
FNR = 5	NR = 5
FNR = 1	NR = 6
FNR = 2	NR = 7
FNR = 3	NR = 8
FNR = 4	NR = 9
FNR = 5	NR = 10

# OFMT : 此变量表示数值输出的格式，它的默认值为 %.6g。
$ awk 'BEGIN {print "OFMT = " OFMT}'
OFMT = %.6g

# OFS : 此变量表示输出域之间的分割符，其默认为空格。
$ awk 'BEGIN {print "OFS = " OFS}' | cat -vte

# 这里^I就是我们test.txt的分隔符：制表符\t
$ awk 'BEGIN {print "OFS = " OFS} {print}' test.txt | cat -vte
OFS =  $
1^Ibirdben^I^Ibejing^I^I28$
2^Ierhuo^I^Ishanghai^I30$
3^Izhangsan^Ishanghai^I20$
4^Ilisi^I^Ishenzhen^I25$
5^Iwangwu^I^Ibeijing^I^I28$

# ORS : 此变量表示输出记录（行）之间的分割符，其默认值是换行符。
$ awk 'BEGIN {print "ORS = " ORS}' | cat -vte
ORS = $
$

# RLENGTH : 此变量表示 match 函数匹配的字符串长度。AWK 的 match 函数用于在输入的字符串中搜索指定字符串。
$ awk 'BEGIN { if (match("One Two Three", "re")) { print RLENGTH } }'

# RS : 此变量表示输入记录的分割符，其默认值为换行符。
$ awk 'BEGIN {print "RS = " RS}' | cat -vte
RS = $
$

# RSTART : 此变量表示由 match 函数匹配的字符串的第一个字符的位置。从1开始。
$ awk 'BEGIN { if (match("One Two Three", "Thre")) { print RSTART } }'

# SUBSEP : 此变量表示数组下标的分割行符，其默认值为 \034 。
$ awk 'BEGIN { print "SUBSEP = " SUBSEP }' | cat -vte
SUBSEP = ^\$

# $0 : 此变量表示整个输入记录。
$ awk '{print $0}' test.txt
1	birdben		bejing		28
2	erhuo		shanghai	30
3	zhangsan	shanghai	20
4	lisi		shenzhen	25
5	wangwu		beijing		28

# $n : 此变量表示当前输入记录的第 n 个域，这些域之间由 FS 分割。
$ awk '{print $1 "\t" $2}' test.txt
1	birdben
2	erhuo
3	zhangsan
4	lisi
5	wangwu
```

### gawk内置变量

```
# ARGIND : 此变量表示当前文件中正在处理的 ARGV 数组的索引值。
$ gawk '{
	print "ARGIND = " ARGIND "\t" "FileName = " ARGV[ARGIND]
}' test.txt test1.txt
ARGIND = 1	FileName = test.txt
ARGIND = 1	FileName = test.txt
ARGIND = 1	FileName = test.txt
ARGIND = 1	FileName = test.txt
ARGIND = 1	FileName = test.txt
ARGIND = 2	FileName = test1.txt
ARGIND = 2	FileName = test1.txt
ARGIND = 2	FileName = test1.txt
ARGIND = 2	FileName = test1.txt
ARGIND = 2	FileName = test1.txt

# IGNORECASE : 当此变量被设置后，GAWK将变得大小写不敏感。
$ gawk 'BEGIN{IGNORECASE=1} /BIRDBEN/' test.txt
1	birdben		bejing		28

# LINT : 此变量提供了在 GAWK 程序中动态控制 --lint 选项的一种途径。当这个变量被设置后， GAWK 会输出 lint 警告信息。如果给此变量赋予字符值 fatal，lint 的所有警告信息将会变了致命错误信息(fatal errors)输出，这和 --lint=fatal 效果一样。
# 设置LINT级别后，会检查awk语法并根据LINT设置的级别给出相应的提示信息
$ gawk 'BEGIN {LINT=1; a}'
gawk: cmd. line:1: warning: reference to uninitialized variable `a'
gawk: cmd. line:1: warning: statement has no effect
```

### gawk选项

```
# --dump-variables[=file] : 该选项会输出排好序的全局变量列表和它们最终的值到文件中，默认的文件是 awkvars.out
$ gawk -v bird=birdben --dump-variables=bird_var.out 'BEGIN {print "bird="bird}'
bird=birdben

$ cat bird_var.out
ARGC: 1
ARGIND: 0
ARGV: array, 1 elements
BINMODE: 0
CONVFMT: "%.6g"
ENVIRON: array, 36 elements
ERRNO: ""
FIELDWIDTHS: ""
FILENAME: ""
FNR: 0
FPAT: "[^[:space:]]+"
FS: " "
FUNCTAB: array, 41 elements
IGNORECASE: 0
LINT: 0
NF: 0
NR: 0
OFMT: "%.6g"
OFS: " "
ORS: "\n"
PREC: 53
PROCINFO: array, 31 elements
RLENGTH: 0
ROUNDMODE: "N"
RS: "\n"
RSTART: 0
RT: ""
SUBSEP: "\034"
SYMTAB: array, 29 elements
TEXTDOMAIN: "messages"
bird: "birdben"

# --profile[=file] : 该选项会输出一份格式化之后的程序到文件中，默认文件是 awkprof.out
$ gawk --profile=bird_profile.out -v bird=birdben 'BEGIN {print "bird="bird}'
bird=birdben

$ cat bird_profile.out
# gawk profile, created Sat Mar  4 13:25:49 2017
# BEGIN rule(s)
BEGIN {
 1  	print "bird=" bird
}
```

### awk条件判断

```
# if-else条件判断
if (condition)
    action-1
else
    action-2
```

### awk循环用法

```
# for循环语法
for (initialisation; condition; increment/decrement)
    action

# while循环语法
while (condition)
    action

# do-while循环语法
do
    action
while (condition)

# Break : 用以结束循环过程。

# Continue : 用于在循环体内部结束本次循环，从而直接进入下一次循环迭代。

# Exit : 用于结束脚本程序的执行。

# Next : 用于跳过你所提供的所有剩下的模式和表达式，直接处理下一个输入行，帮助你阻止运行命令执行过程中多余的步骤。一般配合if-else使用。
```

### awk内置函数

```
# 字符串函数
asort(arr [, d [, how] ]) : asort 函数使用 GAWK 值比较的一般规则排序 arr 中的内容，然后用以 1 开始的有序整数替换排序内容的索引。
asorti(arr [, d [, how] ]) : asorti 函数的行为与 asort 函数的行为很相似，二者的差别在于 aosrt 对数组的值排序，而 asorti 对数组的索引排序。
gsub(regex, sub, string) : gsub 是全局替换( global substitution )的缩写。它将出现的子串（sub）替换为 regx。第三个参数 string 是可选的，默认值为 $0，表示在整个输入记录中搜索子串。
index(str, sub) : index 函数用于检测字符串 sub 是否是 str 的子串。如果 sub 是 str 的子串，则返回子串 sub 在字符串 str 的开始位置；若不是其子串，则返回 0。str 的字符位置索引从 1 开始计数。
length(str) : length 函数返回字符串的长度。
match(str, regex) : match 返回正则表达式在字符串 str 中第一个最长匹配的位置。如果匹配失败则返回0。
split(str, arr, regex) : split 函数使用正则表达式 regex 分割字符串 str。分割后的所有结果存储在数组 arr 中。如果没有指定 regex 则使用 FS 切分。
sprintf(format, expr-list) : sprintf 函数按指定的格式（ format ）将参数列表 expr-list 构造成字符串然后返回。
strtonum(str) : strtonum 将字符串 str 转换为数值。 如果字符串以 0 开始，则将其当作十进制数；如果字符串以 0x 或 0X 开始，则将其当作十六进制数；否则，将其当作浮点数。
sub(regex, sub, string) : sub 函数执行一次子串替换。它将第一次出现的子串用 regex 替换。第三个参数是可选的，默认为 $0。
substr(str, start, l) : substr 函数返回 str 字符串中从第 start 个字符开始长度为 l 的子串。如果没有指定 l 的值，返回 str 从第 start 个字符开始的后缀子串。
tolower(str) : 此函数将字符串 str 中所有大写字母转换为小写字母然后返回。注意，字符串 str 本身并不被改变。
toupper(str) : 此函数将字符串 str 中所有小写字母转换为大写字母然后返回。注意，字符串 str 本身不被改变。

# 时间函数
systime : 此函数返回从 Epoch 以来到当前时间的秒数（在 POSIX 系统上，Epoch 为1970-01-01 00:00:00 UTC）。
mktime(datespec) : 此函数将字符串 dataspec 转换为与 systime 返回值相似的时间戳。 dataspec 字符串的格式为 YYYY MM DD HH MM SS。
strftime([format [, timestamp[, utc-flag]]]) : 此函数根据 format 指定的格式将时间戳 timestamp 格式化。
```

### awk基本用法

示例test.txt文件内容

```
1	birdben		bejing		28
2	erhuo		shanghai	30
3	zhangsan	shanghai	20
4	lisi		shenzhen	25
5	wangwu		beijing		28
```

```
# 输出test.txt文件中的内容
$ awk '{print}' test.txt
$ awk '{print $0}' test.txt
1	birdben		bejing		28
2	erhuo		shanghai	30
3	zhangsan	shanghai	20
4	lisi		shenzhen	25
5	wangwu		beijing		28

# 输出test.txt文件中的指定列的内容
$ awk '{print $2}' test.txt
birdben
erhuo
zhangsan
lisi
wangwu

$ awk '{print $1 "\t" $2}' test.txt
1	birdben
2	erhuo
3	zhangsan
4	lisi
5	wangwu

# 输出test.txt文件中匹配的内容，下面两种方式是等价的
$ awk '/birdben/' test.txt
$ awk '/birdben/ {print}' test.txt
1	birdben		bejing		28

# 输出test.txt文件中匹配的指定列的内容
$ awk '/birdben/ {print $1 "\t" $2}' test.txt
1	birdben

# 在最后输出test.txt文件中匹配的行数
$ awk '/birdben/{++matchCount} END {print "matchCount="matchCount}' test.txt
matchCount=1

# 添加列头然后输出
$ awk 'BEGIN {printf "No\tName\t\t\City\t\tAge\n"} {print}' test.txt
$ awk 'BEGIN {print "No\tName\t\t\City\t\tAge"} {print}' test.txt
No	Name		City		Age
1	birdben		bejing		28
2	erhuo		shanghai	30
3	zhangsan	shanghai	20
4	lisi		shenzhen	25
5	wangwu		beijing		28

# 输出字符超过20的内容
$ awk 'length($0) > 20' test.txt
$ awk 'length($0) > 20 {print}' test.txt
1	birdben		bejing		28
3	zhangsan	shanghai	20
5	wangwu		beijing		28
```

参考文章：

- http://wiki.jikexueyuan.com/project/awk/
