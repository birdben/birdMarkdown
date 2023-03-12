---
title: "Maven总结（二）Maven构建可执行的jar包并且包含依赖jar包"
date: 2016-07-19 02:02:11
tags: [Maven配置]
categories: [Maven]
---

### 插件总结
上一篇我们介绍了Maven如何构建可执行的jar包，主要使用了maven-jar-plugin和maven-dependency-plugin这两个Maven的插件，maven-jar-plugin负责构建jar包，maven-dependency-plugin负责导出所有依赖的jar包，这里我们使用了maven-assembly-plugin插件，这个插件可以将所有当前jar依赖的所有jar包打包到当前jar里面，这样就不需要导出所有依赖的jar包了，直接运行jar就可以了，实在太方便了 ^_^

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

![构建之前对应目录结构](http://img.blog.csdn.net/20170701173706129?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvYmlyZGJlbg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

##### 构建之后对应目录结构

![构建之后对应目录结构](http://img.blog.csdn.net/20170701173744125?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvYmlyZGJlbg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

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

##### 终端
```
# 构建jar包
$ mvn clean compile package

# 进入target目录
$ cd target

# 运行jar
$ java -jar App.jar
SLF4J: Failed to load class "org.slf4j.impl.StaticLoggerBinder".
SLF4J: Defaulting to no-operation (NOP) logger implementation
SLF4J: See http://www.slf4j.org/codes.html#StaticLoggerBinder for further details.
Hello World!
```

参考文章：

- http://chenzhou123520.iteye.com/blog/1706242