---
title: "Maven总结（三）Maven构建可执行的jar包并且读取配置文件"
date: 2017-07-01 16:15:27
tags: [Maven配置]
categories: [Maven]
---

上一篇我们介绍了Maven如何构建可执行的jar包，并且使用Maven插件构建的时候可以将所有当前项目依赖的所有jar包打包到当前jar里面，直接运行jar就可以了。但是如果项目中使用了配置文件，在运行jar的时候应该如何读取相关配置文件呢？这里我们简单的使用log4j.properties文件配置项目日志输出，并且在Maven打包的时候可以指定dev，test，product参数指定使用不同环境的配置文件。

### Maven的pom.xml配置文件

```
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <groupId>com.birdben</groupId>
  <artifactId>birdDemo</artifactId>
  <version>1.0-SNAPSHOT</version>
  <packaging>jar</packaging>

  <name>birdDemo</name>
  <url>http://maven.apache.org</url>

  <properties>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    <!-- 定义配置文件目录 -->
    <profiles.dir>src/main/resources</profiles.dir>
  </properties>

  <dependencies>
    <dependency>
      <groupId>org.slf4j</groupId>
      <artifactId>slf4j-log4j12</artifactId>
      <version>1.7.21</version>
    </dependency>

    <!-- 其他jar包引用开始 -->
    <dependency>
      <groupId>junit</groupId>
      <artifactId>junit</artifactId>
      <version>3.8.1</version>
    </dependency>
    <!-- 其他jar包引用j结束 -->

  </dependencies>

  <!-- 定义不同环境的配置文件目录 -->
  <profiles>
    <profile>
      <id>dev</id>
      <properties>
        <profile.dir>${profiles.dir}/dev</profile.dir>
      </properties>
      <activation>
        <activeByDefault>true</activeByDefault>
      </activation>
    </profile>
    <profile>
      <id>test</id>
      <properties>
        <profile.dir>${profiles.dir}/test</profile.dir>
      </properties>
    </profile>
    <profile>
      <id>product</id>
      <properties>
        <profile.dir>${profiles.dir}/product</profile.dir>
      </properties>
    </profile>
  </profiles>

  <build>
    <finalName>App</finalName>
    <resources>
      <resource>
        <!-- 使用前面定义的${profile.dir}参数指定的配置文件所在的目录 -->
        <directory>${profile.dir}</directory>
        <filtering>true</filtering>
        <targetPath>${project.build.directory}/resources</targetPath>
      </resource>
    </resources>
    <plugins>
      <plugin>
        <artifactId>maven-assembly-plugin</artifactId>
        <configuration>
          <appendAssemblyId>false</appendAssemblyId>
          <descriptorRefs>
            <descriptorRef>jar-with-dependencies</descriptorRef>
          </descriptorRefs>
          <archive>
            <manifest>
              <mainClass>com.birdben.App</mainClass>
            </manifest>
            <!-- 给清单文件添加键值对(配置文件外置) -->
            <manifestEntries>
              <Class-Path>resources/</Class-Path>
            </manifestEntries>
          </archive>
        </configuration>
        <executions>
          <execution>
            <id>make-assembly</id>
            <phase>package</phase>
            <goals>
              <goal>assembly</goal>
            </goals>
          </execution>
        </executions>
      </plugin>
    </plugins>
  </build>
</project>
```

##### 构建之前对应目录结构

![构建之前对应目录结构](http://img.blog.csdn.net/20170701185321169?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvYmlyZGJlbg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

##### 构建之后对应目录结构

![构建之后对应目录结构](http://img.blog.csdn.net/20170701185340987?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvYmlyZGJlbg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

##### App.java

```
package com.birdben;

import org.apache.log4j.PropertyConfigurator;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

/**
 * Hello world!
 */
public class App {

    public static final Logger logger = LoggerFactory.getLogger(App.class);

    public static void main(String[] args) {
        PropertyConfigurator.configure("resources/log4j.properties");
        logger.info("App start");
        System.out.println("Hello World!");
        logger.info("App end");
    }
}
```

##### log4j.properties

```
# Default to info level output; this is very handy if you eventually use Hibernate as well.
log4j.rootCategory=info,console

log4j.additivity.org.apache=true
# 控制台(console)
log4j.appender.console=org.apache.log4j.DailyRollingFileAppender
log4j.appender.console.Threshold=DEBUG
log4j.appender.console.encoding=UTF-8
log4j.appender.console.ImmediateFlush=true
log4j.appender.console.Append=true
log4j.appender.console.File=logs/test.log
log4j.appender.console.layout=org.apache.log4j.PatternLayout
log4j.appender.console.layout.ConversionPattern=%d{yyyy-MM-dd HH:mm:ss} [%5p] - %c -%F(%L) -%m%n

# 日志文件(logFile)
log4j.appender.logFile=org.apache.log4j.DailyRollingFileAppender
log4j.appender.logFile.Threshold=DEBUG
log4j.appender.logFile.encoding=UTF-8
log4j.appender.logFile.ImmediateFlush=true
log4j.appender.logFile.Append=true
log4j.appender.logFile.File=logs/test.log
log4j.appender.logFile.layout=org.apache.log4j.PatternLayout
log4j.appender.logFile.layout.ConversionPattern=%d{yyyy-MM-dd HH:mm:ss} [%5p] - %c -%F(%L) -%m%n
```

##### 终端

```
# 构建jar包，-P参数之后指定使用的env环境（这里使用test的配置）
$ mvn -Ptest clean compile package

# 进入target目录
$ cd target

# 运行jar
$ java -jar App.jar
Hello World!

# 查看日志文件
$ cat logs/test.log
2017-07-01 18:55:57 [ INFO] - com.birdben.App -App.java(16) -App start
2017-07-01 18:55:57 [ INFO] - com.birdben.App -App.java(18) -App end
```

参考文章：

- https://stackoverflow.com/questions/7421612/slf4j-failed-to-load-class-org-slf4j-impl-staticloggerbinder
- http://blog.csdn.net/e_wsq/article/details/64537091
- http://www.cnblogs.com/hdwang/p/5418747.html
