---
title: "JVM常用命令学习——jinfo的使用"
date: 2017-07-30 15:07:59
tags: [JVM]
categories: [Java]
---

### jinfo命令

jinfo(Java Configuration Information)，主要用于查看指定Java进程(或核心文件、远程调试服务器)的Java配置信息。

```
$ jinfo --help
Usage:
    jinfo [option] <pid>
        (to connect to running process)
    jinfo [option] <executable <core>
        (to connect to a core file)
    jinfo [option] [server_id@]<remote server IP or hostname>
        (to connect to remote debug server)

where <option> is one of:
    -flag <name>         to print the value of the named VM flag
    -flag [+|-]<name>    to enable or disable the named VM flag
    -flag <name>=<value> to set the named VM flag to the given value
    -flags               to print VM flags
    -sysprops            to print Java system properties
    <no option>          to print both of the above
    -h | -help           to print this help message
```

通过help提示可以看出基本的命令输入格式为：

```
jinfo [option] <pid> : 指定进程号(pid)的进程jinfo [option] <executable <core> : 指定核心文件
jinfo [option] [server_id@]<remote server IP or hostname> : 指定远程调试服务器

参数：

<pid> : 需要打印配置信息的进程ID。
<executable> : 产生核心dump的Java可执行文件。
<core> : 需要打印配置信息的核心文件。
<remote-hostname-or-IP/hostname> : 远程调试服务器的主机名或IP地址。
<server-id> : 可选的唯一id，如果相同的远程主机上运行了多台调试服务器，用此选项参数标识服务器。

[option]参数：

<no option> : 打印命令行标识参数和系统属性键值对。
-flag <name> : 打印指定的命令行标识参数的名称和值。
-flag [+|-]<name> : 启用或禁用指定的boolean类型的命令行标识参数。
-flag <name>=<value> : 为给定的命令行标识参数设置指定的值。
-flags : 成对打印传递给JVM的命令行标识参数。
-sysprops : 以键值对形式打印Java系统属性。
-h : 打印帮助信息。
-help : 打印帮助信息。
```

### 实例

```
$ jinfo -flags 2699
Attaching to process ID 2699, please wait...
Debugger attached successfully.
Server compiler detected.
JVM version is 25.121-b13
Non-default VM flags: -XX:CICompilerCount=4 -XX:InitialHeapSize=268435456 -XX:MaxHeapSize=4294967296 -XX:MaxNewSize=1431306240 -XX:MinHeapDeltaBytes=524288 -XX:NewSize=89128960 -XX:OldSize=179306496 -XX:+UseCompressedClassPointers -XX:+UseCompressedOops -XX:+UseFastUnorderedTimeStamps -XX:+UseParallelGC
Command line:  -Djava.util.logging.config.file=/Users/yunyu/apache-tomcat-7.0.67/conf/logging.properties -Djava.util.logging.manager=org.apache.juli.ClassLoaderLogManager -Djava.endorsed.dirs=/Users/yunyu/apache-tomcat-7.0.67/endorsed -Dcatalina.base=/Users/yunyu/apache-tomcat-7.0.67 -Dcatalina.home=/Users/yunyu/apache-tomcat-7.0.67 -Djava.io.tmpdir=/Users/yunyu/apache-tomcat-7.0.67/temp
```

```
$ jinfo 2699
Attaching to process ID 2699, please wait...
Debugger attached successfully.
Server compiler detected.
JVM version is 25.121-b13
Java System Properties:

java.runtime.name = Java(TM) SE Runtime Environment
java.vm.version = 25.121-b13
sun.boot.library.path = /Library/Java/JavaVirtualMachines/jdk1.8.0_121.jdk/Contents/Home/jre/lib
shared.loader =
gopherProxySet = false
java.vendor.url = http://java.oracle.com/
java.vm.vendor = Oracle Corporation
path.separator = :
file.encoding.pkg = sun.io
java.vm.name = Java HotSpot(TM) 64-Bit Server VM
java.util.logging.config.file = /Users/yunyu/apache-tomcat-7.0.67/conf/logging.properties
tomcat.util.buf.StringCache.byte.enabled = true
sun.os.patch.level = unknown
sun.java.launcher = SUN_STANDARD
user.country = CN
user.dir = /Users/yunyu/apache-tomcat-7.0.67/bin
java.vm.specification.name = Java Virtual Machine Specification
java.runtime.version = 1.8.0_121-b13
org.apache.catalina.startup.TldConfig.jarsToSkip = tomcat7-websocket.jar
java.awt.graphicsenv = sun.awt.CGraphicsEnvironment
os.arch = x86_64
java.endorsed.dirs = /Users/yunyu/apache-tomcat-7.0.67/endorsed
line.separator =

java.io.tmpdir = /Users/yunyu/apache-tomcat-7.0.67/temp
java.vm.specification.vendor = Oracle Corporation
java.util.logging.manager = org.apache.juli.ClassLoaderLogManager
java.naming.factory.url.pkgs = org.apache.naming
os.name = Mac OS X
sun.jnu.encoding = UTF-8
java.library.path = /Users/yunyu/Library/Java/Extensions:/Library/Java/Extensions:/Network/Library/Java/Extensions:/System/Library/Java/Extensions:/usr/lib/java:.
tomcat.util.scan.DefaultJarScanner.jarsToSkip = bootstrap.jar,commons-daemon.jar,tomcat-juli.jar,annotations-api.jar,el-api.jar,jsp-api.jar,servlet-api.jar,websocket-api.jar,catalina.jar,catalina-ant.jar,catalina-ha.jar,catalina-tribes.jar,jasper.jar,jasper-el.jar,ecj-*.jar,tomcat-api.jar,tomcat-util.jar,tomcat-coyote.jar,tomcat-dbcp.jar,tomcat-jni.jar,tomcat-spdy.jar,tomcat-i18n-en.jar,tomcat-i18n-es.jar,tomcat-i18n-fr.jar,tomcat-i18n-ja.jar,tomcat-juli-adapters.jar,catalina-jmx-remote.jar,catalina-ws.jar,tomcat-jdbc.jar,tools.jar,commons-beanutils*.jar,commons-codec*.jar,commons-collections*.jar,commons-dbcp*.jar,commons-digester*.jar,commons-fileupload*.jar,commons-httpclient*.jar,commons-io*.jar,commons-lang*.jar,commons-logging*.jar,commons-math*.jar,commons-pool*.jar,jstl.jar,taglibs-standard-spec-*.jar,geronimo-spec-jaxrpc*.jar,wsdl4j*.jar,ant.jar,ant-junit*.jar,aspectj*.jar,jmx.jar,h2*.jar,hibernate*.jar,httpclient*.jar,jmx-tools.jar,jta*.jar,log4j.jar,log4j-1*.jar,mail*.jar,slf4j*.jar,xercesImpl.jar,xmlParserAPIs.jar,xml-apis.jar,junit.jar,junit-*.jar,hamcrest*.jar,org.hamcrest*.jar,ant-launcher.jar,cobertura-*.jar,asm-*.jar,dom4j-*.jar,icu4j-*.jar,jaxen-*.jar,jdom-*.jar,jetty-*.jar,oro-*.jar,servlet-api-*.jar,tagsoup-*.jar,xmlParserAPIs-*.jar,xom-*.jar
java.class.version = 52.0
java.specification.name = Java Platform API Specification
sun.management.compiler = HotSpot 64-Bit Tiered Compilers
os.version = 10.11.5
user.home = /Users/yunyu
org.apache.catalina.startup.ContextConfig.jarsToSkip =
user.timezone = Asia/Shanghai
catalina.useNaming = true
java.awt.printerjob = sun.lwawt.macosx.CPrinterJob
file.encoding = UTF-8
java.specification.version = 1.8
catalina.home = /Users/yunyu/apache-tomcat-7.0.67
user.name = yunyu
java.class.path = /Users/yunyu/apache-tomcat-7.0.67/bin/bootstrap.jar:/Users/yunyu/apache-tomcat-7.0.67/bin/tomcat-juli.jar
java.naming.factory.initial = org.apache.naming.java.javaURLContextFactory
package.definition = sun.,java.,org.apache.catalina.,org.apache.coyote.,org.apache.jasper.,org.apache.naming.,org.apache.tomcat.
java.vm.specification.version = 1.8
sun.arch.data.model = 64
sun.java.command = org.apache.catalina.startup.Bootstrap start
java.home = /Library/Java/JavaVirtualMachines/jdk1.8.0_121.jdk/Contents/Home/jre
user.language = zh
java.specification.vendor = Oracle Corporation
awt.toolkit = sun.lwawt.macosx.LWCToolkit
java.vm.info = mixed mode
java.version = 1.8.0_121
java.ext.dirs = /Users/yunyu/Library/Java/Extensions:/Library/Java/JavaVirtualMachines/jdk1.8.0_121.jdk/Contents/Home/jre/lib/ext:/Library/Java/Extensions:/Network/Library/Java/Extensions:/System/Library/Java/Extensions:/usr/lib/java
sun.boot.class.path = /Library/Java/JavaVirtualMachines/jdk1.8.0_121.jdk/Contents/Home/jre/lib/resources.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_121.jdk/Contents/Home/jre/lib/rt.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_121.jdk/Contents/Home/jre/lib/sunrsasign.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_121.jdk/Contents/Home/jre/lib/jsse.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_121.jdk/Contents/Home/jre/lib/jce.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_121.jdk/Contents/Home/jre/lib/charsets.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_121.jdk/Contents/Home/jre/lib/jfr.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_121.jdk/Contents/Home/jre/classes
server.loader =
java.vendor = Oracle Corporation
catalina.base = /Users/yunyu/apache-tomcat-7.0.67
file.separator = /
java.vendor.url.bug = http://bugreport.sun.com/bugreport/
common.loader = ${catalina.base}/lib,${catalina.base}/lib/*.jar,${catalina.home}/lib,${catalina.home}/lib/*.jar
sun.io.unicode.encoding = UnicodeBig
sun.cpu.endian = little
package.access = sun.,org.apache.catalina.,org.apache.coyote.,org.apache.jasper.,org.apache.naming.resources.,org.apache.tomcat.
sun.cpu.isalist =

VM Flags:
Non-default VM flags: -XX:CICompilerCount=4 -XX:InitialHeapSize=268435456 -XX:MaxHeapSize=4294967296 -XX:MaxNewSize=1431306240 -XX:MinHeapDeltaBytes=524288 -XX:NewSize=89128960 -XX:OldSize=179306496 -XX:+UseCompressedClassPointers -XX:+UseCompressedOops -XX:+UseFastUnorderedTimeStamps -XX:+UseParallelGC
Command line:  -Djava.util.logging.config.file=/Users/yunyu/apache-tomcat-7.0.67/conf/logging.properties -Djava.util.logging.manager=org.apache.juli.ClassLoaderLogManager -Djava.endorsed.dirs=/Users/yunyu/apache-tomcat-7.0.67/endorsed -Dcatalina.base=/Users/yunyu/apache-tomcat-7.0.67 -Dcatalina.home=/Users/yunyu/apache-tomcat-7.0.67 -Djava.io.tmpdir=/Users/yunyu/apache-tomcat-7.0.67/temp
```

我这里使用jinfo查看本地的tomcat进程，可以看到打印出的信息以Key-Value的形式显示相关信息。

VM Flags显示了tomcat的一些JVM信息：

- -XX:CICompilerCount=4：设置最大并行编译数为4
- -XX:InitialHeapSize=268435456：是初始堆内存大小为256M
- -XX:MaxHeapSize=4294967296：是最大堆内存大小为4096M
- -XX:MaxNewSize=1431306240：是年轻代最大值为1365M
- -XX:MinHeapDeltaBytes：是GC最小改变堆内存值为0.5M
- -XX:NewSize=89128960：是初始年轻代大小为85M
- -XX:OldSize=179306496：是初始年老代大小为171M
- -XX:+UseCompressedClassPointers：开启压缩类指针
- -XX:+UseCompressedOops：开启压缩对象指针
- -XX:+UseFastUnorderedTimeStamps：（没弄懂是什么意思）
- -XX:+UseParallelGC：开启此参数使用Parallel Scavenge & Parallel Old的组合进行GC回收。

参考文章：

- http://www.softown.cn/post/182.html