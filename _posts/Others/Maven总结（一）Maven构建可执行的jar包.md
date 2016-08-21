---
title: "Maven总结（一）Maven构建可执行的jar包"
date: 2016-07-07 05:48:09
tags: [Maven配置]
categories: [Maven]
---

### 步骤
1. 需要指定的可执行的class，需要有入口main方法
2. 使用mvn package来构建jar包
3. 通过java -jar App.jar来执行jar包

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
  </properties>

  <dependencies>
    <dependency>
      <groupId>org.slf4j</groupId>
      <artifactId>slf4j-api</artifactId>
      <version>1.7.13</version>
      <scope>system</scope>
      <!-- ${basedir} 项目根目录 -->
      <systemPath>${basedir}/lib/slf4j-api-1.7.13.jar</systemPath>
    </dependency>

    <!-- 其他jar包引用开始 -->
    <dependency>
      <groupId>junit</groupId>
      <artifactId>junit</artifactId>
      <version>3.8.1</version>
    </dependency>
    <!-- 其他jar包引用j结束 -->

  </dependencies>

  <build>
    <finalName>App</finalName>
    <plugins>
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-jar-plugin</artifactId>
        <version>2.4</version>
        <configuration>
          <archive>
            <manifest>
              <!-- 指定Main方法入口的class -->
              <mainClass>com.birdben.App</mainClass>
              <!-- 在jar包的MANIFEST.MF文件中生成Class-Path属性 -->
              <addClasspath>true</addClasspath>
              <!-- Class-Path 前缀 -->
              <classpathPrefix>lib/</classpathPrefix>
            </manifest>
            <!-- 把本地的第三方jar包添加到MANIFEST.MF文件中,可以解压打包后的jar包查看MANIFEST.MF文件 -->
            <!--
            Class-Path: 指定当前jar包执行所依赖的classpath,包括本地的第三方jar包和maven引入的jar包
            Class-Path: lib/slf4j-api-1.7.13.jar lib/junit-3.8.1.jar
            Main-Class: 指定当前jar包的入口class
            Main-Class: com.birdben.App
            -->
            <manifestEntries>
              <Class-Path>lib/slf4j-api-1.7.13.jar</Class-Path>
            </manifestEntries>
          </archive>
        </configuration>
      </plugin>
      <!-- 拷贝依赖的jar包到lib目录 -->
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-dependency-plugin</artifactId>
        <executions>
          <execution>
            <id>copy</id>
            <phase>package</phase>
            <goals>
              <goal>copy-dependencies</goal>
            </goals>
            <configuration>
              <!-- ${project.build.directory} 构建目录，缺省为target -->
              <outputDirectory>
                ${project.build.directory}/lib
              </outputDirectory>
            </configuration>
          </execution>
        </executions>
      </plugin>
    </plugins>
  </build>
</project>
```

##### App.java
```
package com.birdben;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import junit.framework.TestResult;

/**
 * Hello world!
 */
public class App {

    public static Logger logger = LoggerFactory.getLogger(App.class);

    public static void main(String[] args) {
        TestResult test = new TestResult();
        logger.info("App start");
        System.out.println("Hello World!");
        logger.info("App end");
    }
}
```

##### MANIFEST.MF
```
Manifest-Version: 1.0
Archiver-Version: Plexus Archiver
Built-By: ben
Class-Path: lib/slf4j-api-1.7.13.jar lib/junit-3.8.1.jar
Created-By: Apache Maven 3.3.1
Build-Jdk: 1.8.0_65
Main-Class: com.birdben.App
```

##### 终端
```
# 运行jar
$ java -jar App.jar
```

##### 遇到问题和解决方法

- Q1 : 执行java -jar App.jar，报错：找不到或无法加载主类XXX.XXXX.XXX
- A1 : 添加CLASSPATH环境变量

```
export CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
```

- Q2 : 执行java -jar App.jar，报错：Exception in thread "main" java.lang.NoClassDefFoundError: org/slf4j/LoggerFactory
	at com.birdben.App.<clinit>(App.java:11)
Caused by: java.lang.ClassNotFoundException: org.slf4j.LoggerFactory
	at java.net.URLClassLoader.findClass(URLClassLoader.java:381)
	at java.lang.ClassLoader.loadClass(ClassLoader.java:424)
	at sun.misc.Launcher$AppClassLoader.loadClass(Launcher.java:331)
	at java.lang.ClassLoader.loadClass(ClassLoader.java:357)
	... 1 more
	
- A2 : 是因为执行jar的时候没有找到对应的依赖的第三方jar包，在打好的jar包中的MANIFEST.MF文件的Class-Path也未找到第三方jar包的路径。解决方法就是在<manifestEntries>中添加本地的第三方jar包到MANIFEST.MF文件中。

```
<!-- 把本地的第三方jar包添加到MANIFEST.MF文件中,可以解压打包后的jar包查看MANIFEST.MF文件 -->
<!--
Class-Path: 指定当前jar包执行所依赖的classpath,包括本地的第三方jar包和maven引入的jar包
Class-Path: lib/slf4j-api-1.7.13.jar lib/junit-3.8.1.jar
Main-Class: 指定当前jar包的入口class
Main-Class: com.birdben.App
-->
<manifestEntries>
  <Class-Path>lib/slf4j-api-1.7.13.jar</Class-Path>
</manifestEntries>
```

参考文章：

- http://www.tuicool.com/articles/rQFnmu
- http://my.oschina.net/zimingforever/blog/266191
- http://www.cnblogs.com/hdwang/p/5418747.html