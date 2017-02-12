---
title: "Go学习（一）开发环境搭建"
date: 2016-11-29 15:08:30
tags: [Go]
categories: [Go]
---

我这里是Mac系统，不同的操作系统安装略有不同，请知晓

### Go安装
```
# 我这里直接使用homebrew安装Go，非常的简单
$ brew install go

# 安装成功后，查看Go的环境
$ go env
GOARCH="amd64"
GOBIN=""
GOEXE=""
GOHOSTARCH="amd64"
GOHOSTOS="darwin"
GOOS="darwin"
GOPATH=""
GORACE=""
GOROOT="/usr/local/Cellar/go/1.7.3/libexec"
GOTOOLDIR="/usr/local/Cellar/go/1.7.3/libexec/pkg/tool/darwin_amd64"
CC="clang"
GOGCCFLAGS="-fPIC -m64 -pthread -fno-caret-diagnostics -Qunused-arguments -fmessage-length=0 -fdebug-prefix-map=/var/folders/0h/jtjrr7g95mv2pt4ts1tgmzyh0000gn/T/go-build273829653=/tmp/go-build -gno-record-gcc-switches -fno-common"
CXX="clang++"
CGO_ENABLED="1"
```

### 配置环境变量
```
GOPATH=/Users/yunyu/workspace_go
GOROOT=/usr/local/Cellar/go/1.7.3/libexec

export GOPATH
export GOROOT
```

在我们以前熟悉的各种语言中都有这样几个概念：系统路径,官方包路径,第三方包路径,项目路径。

Go中只有两个路径：

- GOROOT: Go的安装路径,官方包路径根据这个设置自动匹配
- GOPATH: 工作路径(其实不应该用中文翻译解释，直接说GOPATH更合适)

问题：项目路径和第三方包路径呢？ 首先：Go中是没有项目这个概念的，只有包。可执行包只是特殊的一种，类似我们常说的项目GOPATH可以设置多个，不管是可执行包，还是非可执行包，通通都应该在某个$GOPATH/src下。

$GOPATH可以包含多个工作目录，取决于你的个人情况。如果你设置了多个工作目录，那么当你在之后使用 go get（远程包安装命令）时远程包将会被安装在第一个目录下。

以上都可以在Go对于gopath和importpath的help使用说明中查看到

```
$ go help gopath
$ go help importpath
```

### Go的目录结构

以上$GOPATH目录约定有三个子目录：

1. src 存放源代码（比如：.go .c .h .s等）。src 子目录通常包会含多种版本控制的代码仓库（例如Git或Mercurial）， 以此来跟踪一个或多个源码包的开发。
2. pkg 编译后生成的包文件（比如：.a）
3. bin 编译后生成的可执行文件

### 编写helloworld.go

##### helloworld.go

```
package main

//引入fmt库
import "fmt"

func main() {
    fmt.Println("Hello World!")
}
```

### 程序入口点(entry point)和包(package)

Go保持了与C家族语言一致的风格：即目标为可执行程序的Go源码中务必要有一个名为main的函数，该函数即为可执行程序的入口点。除此之外Go还增加了一个约束：作为入口点的main函数必须在名为main的package中。正如上面helloworld.go源文件中的那样，在源码第一行就声明了该文件所归属的package为main。

Go去除了头文件的概念，而借鉴了很多主流语言都采用的package的源码组织方式。package是个逻辑概念，与文件没有一一对应的关系。 如果多个源文件都在开头声明自己属于某个名为foo的包，那这些源文件中的代码在逻辑上都归属于包foo(这些文件最好在同一个目录下，至少目前的Go版本还无法支持不同目录下的源文件归属于同一个包)。

我们看到helloworld.go中import一个名为fmt的包，并利用该包内的Println函数输出"Hello World!"。直觉告诉我们fmt包似乎是一个标准库中的包。没错，fmt包提供了格式化文本输出以及读取格式化输入的相关函数，与C中的printf或scanf等类似。我们通过import语句将fmt包导入我们的源文件后就可以使用该fmt包导出(export)的功能函数了(比如 Printf)。

在C中，我们通过static来标识局部函数还是全局函数。而在Go中，包中的函数是否可以被外部调用，要看该函数名的首母是否为大写。这是一种Go语言固化的约定：首母大写的函数被认为是导出的函数，可以被包之外的代码调用；而小写字母开头的函数则仅能在包内使用。在例子中你也看到了fmt包的Println函数其首母就是大写的。

##### hello.go

```
package hello

import "fmt"

func Hello() {
    fmt.Println("Hello World!")
}
```

##### main.go

```
package main

import (
    "hello"
)

func main() {
    hello.Hello()
}
```

用go build编译main.go，结果如下：

```
# 执行run或者build都会报错
$ go run main.go
main.go:4:5: cannot find package "hello" in any of:
	/usr/local/Cellar/go/1.7.3/libexec/src/hello (from $GOROOT)
	/Users/yunyu/workspace_go/src/hello (from $GOPATH)

$ go build main.go
main.go:4:5: cannot find package "hello" in any of:
	/usr/local/Cellar/go/1.7.3/libexec/src/hello (from $GOROOT)
	/Users/yunyu/workspace_go/src/hello (from $GOPATH)
```

编译器居然提示无法找到hello这个package，而hello.go中明明定义了package hello了。这是怎么回事呢？原来go compiler搜索package的方式与我们常规理解的有不同，Go在这方面也有一套约定，这里面涉及到一个重要的环境变量：GOPATH。我们可以使用go help gopath来查看一下有关gopath的manual。

Go compiler的package搜索顺序是这样的，以搜索hello这个package为例：

* 首先，Go compiler会在GO安装目录(GOROOT，这里是Mac环境安装路径是/usr/local/Cellar/go/1.7.3/libexec，如果是Linux安装目录可能是/usr/local/go)下查找是否有src/pkg/hello相关包源码；如果没有则继续；
* 如果export GOPATH=PATH1:PAHT2，则Go compiler会依次查找是否存在PATH1/src/hello、PATH2/src/hello；配置在GOPATH中的PATH1和PATH2被称作workplace（类似Java项目中的project）；
* 如果在上述几个位置均无法找到hello这个package，则提示出错。

在本例子中，我设置的GOPATH环境变量有问题，之前把GOPATH理解成了Java中的workspace了，也没有建立类似PATH1/src/hello（workspace_go/src/hello）这样的路径，因此Go compiler显然无法找到hello这个package了。我们来修改一下GOPATH变量并建立相关目录：

修改环境变量如下：

##### /etc/profile或者~/.bash_profile

```
export TEST_PATH=/Users/yunyu/workspace_go/TestGo
export DEMO_PATH=/Users/yunyu/workspace_go/GoDemo
export GOPATH=$TEST_PATH:$DEMO_PATH
```

目录结构如下：

```
workspace_go
     |
     |--- TestGo
     |      |
     |      |--- src
     |            |
     |            |--- hello
     |            |      |
     |            |      |--- hello.go
     |            |
     |            |--- main.go
     |
     |-- GoDemo
```

再次执行run或者build都能正常运行
  
```
$ go run main.go
Hello World!

$ go build main.go
$ ./main
Hello World!
```

我们将main.go移入到main目录下，这样目录结构更加合理。目录结构如下：

```
workspace_go
     |
     |--- TestGo
     |      |
     |      |--- src
     |            |
     |            |--- hello
     |            |      |
     |            |      |--- hello.go
     |            |
     |            |--- main
     |                   |
     |                   |--- main.go
     |
     |-- GoDemo
```

Go提供了install命令，与build命令相比，install命令在编译源码后还会将可执行文件或库文件安装到约定的目录下。我们以main目录为例：

```
$ cd main
$ go install
```

install命令执行后，我们发现main目录下没有任何变化，原先build时产生的main可执行文件也不见了踪影。别急，前面说过Go install也有一套自己的约定：
* go install(在src/DIR下)编译出的可执行文件以其所在目录名(DIR)命名
* go install将可执行文件安装到与src同级别的bin目录下，bin目录由go install自动创建
* go install将可执行文件依赖的各种package编译后，放在与src同级别的pkg目录下

现在我们来看看bin目录：

```
$ ls /Users/yunyu/workspace_go/TestGo
bin
src
pkg

$ cd /Users/yunyu/workspace_go/TestGo
$ ls bin
main
```

的确出现一个bin目录和一个pkg目录，并且刚刚编译的程序main在bin下面。

hello.go编译后并非可执行程序，在编译main的同时，由于main依赖hello package，因此hello也被关联编译了。这与单独在hello目录下执行install的结果是一样的，只是单独安装的时候我们只创建了pkg目录，没有bin目录（因为bin目录是install main.go产生的），我们试试：

```
$ cd /Users/yunyu/workspace_go/TestGo/src/hello
$ go install
$ ls /Users/yunyu/workspace_go/TestGo
bin
pkg
src
```

在我们的workspace(TestGo目录)下出现了一个pkg目录，pkg目录下是一个名为darwin_amd64的子目录(这个目录和操作系统有关系)，其下面有一个文件：hello.a。这就是我们install的结果。hello.go被编译为hello.a并安装到pkg/darwin_amd64目录下了。

```
workspace_go
     |
     |--- TestGo
     |      |
     |      |--- src
     |      |     |
     |      |     |--- hello
     |      |     |      |
     |      |     |      |--- hello.go
     |      |     |
     |      |     |--- main
     |      |            |
     |      |            |--- main.go
     |      |
     |      |--- bin
     |      |     |
     |      |     |--- main
     |      |
     |      |
     |      |--- pkg
     |            |
     |            |--- hello.a
     |
     |-- GoDemo
```

.a这个后缀名让我们想起了静态共享库，但这里的.a却是Go独有的文件格式，与传统的静态共享库并不兼容。但Go语言的设计者使用这个后缀名似乎是希望这个.a文件也承担起Go语言中"静态共享库"的角色。我们不妨来试试，看看这个hello.a是否可以被Go compiler当作"静态共享库"来对待。我们移除src中的hello目录，然后在main目录下执行go build：

```
$ go build
main.go:4:5: cannot find package "hello" in any of:
	/usr/local/Cellar/go/1.7.3/libexec/src/hello (from $GOROOT)
	/Users/yunyu/workspace_go/TestGo/src/hello (from $GOPATH)
```

Go编译器提示无法找到hello这个包，可见目前版本的Go编译器似乎不理pkg下的.a文件。http://code.google.com/p/go/issues/detail?id=2775 这个issue也印证了这一点，不过后续Go版本很可能会支持链接.a文件。毕竟我们在使用第三方package的时候，很可能无法得到其源码，并且在每个项目中都保存一份第三方包的源码也十分不利于项目源码的后期维护。

### 像脚本一样运行Go源码

Go具有很高的编译效率，这得益于其设计者对该目标的重视以及设计过程中细节方面的把控，当然这不是本文要关注的话题。正是由于go具有极速的编译，我们才可以像使用运行脚本语言那样使用它。

目前Go提供了run命令来直接运行源文件。比如：

```
$ go run main.go
Hello World!
```

go run实际上是一个将编译源码和运行编译后的二进制程序结合在一起的命令。但目前go源文件尚不支持作成Shebang Script，因为Go compiler尚不识别#!符号，下面的源码文件运行起来会出错：

```
#! /usr/local/bin/go run

package main

import (
    "hello"
)

func main() {
    hello.Hello()
}

$ go run main.go
package main:
main.go:1:1: illegal character U+0023 '#'
```

不过我们可以可借助一些第三方工具来运行Shebang Go scripts，比如gorun。


### go run， go build， go install的区别

```
go run helloworld.go
go build helloworld.go
go install
```

#### go build

通过go build加上要编译的Go源文件名，我们即可得到一个可执行文件，默认情况下这个文件的名字为源文件名字去掉.go后缀。

```
$ go build helloworld.go
$ ls
helloworld
helloworld.go
```

当然我们也 可以通过-o选项来指定其他名字：

```
$ go build -o newworld helloworld.go
$ ls
newworld
helloworld.go
```

如果我们在当前目录(src)下直接执行go build命令，后面不带文件名，我们将得到一个与目录名同名的可执行文件：

```
$ go build
$ ls
src
helloworld.go
```

#### go install

与build命令相比，install命令在编译源码后还会将可执行文件或库文件安装到约定的目录下。

go install编译出的可执行文件以其所在目录名(DIR)命名
go install将可执行文件安装到与src同级别的bin目录下，bin目录由go install自动创建
go install将可执行文件依赖的各种package编译后，放在与src同级别的pkg目录下。

#### go run

go run实际上是一个将编译源码和运行编译后的二进制程序结合在一起的命令。

go run helloworld.go = go build helloworld.go + ./helloworld


### 引用第三方包

若你在包的导入路径中包含了代码仓库的URL，go get就会自动地获取、构建并安装它：

```
$ go get github.com/golang/glog
```

若指定的包不在工作空间中，go get就会将会将它放到GOPATH指定的第一个工作空间内。（若该包已存在，go get就会跳过远程获取，其行为与go install相同）

在执行完上面的go get命令后，工作空间的目录树看起来应该是这样的：
			
```
workspace_go
     |
     |--- TestGo
     |      |
     |      |--- src
     |      |     |
     |      |     |--- github.com
     |      |     |      |
     |      |     |      |--- golang
     |      |     |             |
     |      |     |             |--- glog
     |      |     |                    |
     |      |     |                    |--- LICENSE
     |      |     |                    |
     |      |     |                    |--- README
     |      |     |                    |
     |      |     |                    |--- glog.go
     |      |     |                    |
     |      |     |                    |--- glog_file.go
     |      |     |                    |
     |      |     |                    |--- glog_test.go
     |      |     |
     |      |     |--- hello
     |      |     |      |
     |      |     |      |--- hello.go
     |      |     |
     |      |     |--- main
     |      |            |
     |      |            |--- main.go
     |      |
     |      |--- bin
     |      |     |
     |      |     |--- main
     |      |
     |      |
     |      |--- pkg
     |            |
     |            |--- github.com
     |            |      |
     |            |      |--- golang
     |            |             |
     |            |             |--- glog.a
     |            |
     |            |--- hello.a
     |
     |-- GoDemo
```

hello.go

```
package hello

import "fmt"
import "flag"
import "github.com/golang/glog"

func Hello() {
    flag.Parse()
    fmt.Println("Hello World!")
    glog.Info("glog Hello World!")
    glog.Flush()
}
```

注意事项:  

1. 因为是使用时依参数配置，所以需在main()中加上flag.Parse()
2. 需在结尾加上glog.Flush()
3. 依级别生成不同的日志文件，但级别高的日志信息会同时在级别低的日志文件中输出

```
$ go run main.go -log_dir=./
Hello World!
$ ll
lrwxr-xr-x  1 yunyu  staff   51 12 27 11:44 main.INFO -> main.localhost.yunyu.log.INFO.20161227-114425.19830
-rw-r--r--  1 yunyu  staff   72 12 22 14:39 main.go
-rw-r--r--  1 yunyu  staff  186 12 27 11:44 main.localhost.yunyu.log.INFO.20161227-114425.19830

$ vi main.INFO
Log file created at: 2016/12/27 11:48:08
Running on machine: localhost
Binary: Built with gc go1.7.3 for darwin/amd64
Log line format: [IWEF]mmdd hh:mm:ss.uuuuuu threadid file:line] msg
I1227 11:48:08.088596   19951 hello.go:10] glog Hello World!
```


参考文章：

- http://tonybai.com/2012/08/17/hello-go/
- http://docscn.studygolang.com/doc/code.html
- http://blog.csdn.net/xcl168/article/details/44650019
- http://www.oschina.net/translate/go-at-google-language-design-in-the-service-of-software-engineering?lang=chs&page=1#