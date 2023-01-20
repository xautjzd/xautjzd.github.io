---
title: Java jar 是如何执行的?
date: "2020-09-01T00:00:00Z"
tags: ["Java"]
---

Java 作为一门古老又流行的编程语言，想必大家或多或少都听过，甚至使用过，也有一部分人甚至在生产环境部署过 java 应用。可你了解过 java 程序是如何加载运行的吗？如果你的答案是 `yes`, 那么建议不用浪费时间花费在已知的知识点了。

下面主要针对 java 可执行 jar(还有有库类型的jar 包，eg: rt.jar) 分析 jar 的加载运行过程，主要包含以下三部分展开:

- Java jar 生成
- Java jar 内容
- Java jar 加载 & 运行

<!--more-->

## Java jar 生成

首先，看下 wikipedia 上对于 [jar](https://en.wikipedia.org/wiki/JAR_(file_format)) 的定义:

> A JAR (Java ARchive) is a package file format typically used to aggregate many Java class files and associated metadata and resources (text, images, etc.) into one file for distribution

jar 只是用于打包 java 应用的一种格式, 可通过 `jar -xf demo.jar` 解压查看 jar 内容。

目前，我们常用的 jar 生成方式:

- [Maven jar plugin](https://maven.apache.org/plugins/maven-jar-plugin/)
- [Gradle java plugin](https://docs.gradle.org/current/userguide/java_plugin.html)
- jdk 自带命令: jar

下面举一个例子说明。

```java
// com/demo/Hello.java
package com.demo;

public class Hello {
    public static void main (String[] args) {
        System.out.println("hello, world");
    }
}
```
执行 `javac com/demo/Hello.java` 进行编译, 会在 `com/demo/` 目录下生成 Hello.class 字节码，再执行: `java cfe hello.jar com.demo.Hello com/demo/Hello.class` 生成可执行 jar，其中 `-e` 指定程序入口(即: main 函数所在类)，可通过: `java -jar hello.jar` 执行 jar。

## Java jar 内容

接着我们可以通过 `jar xvf hello.jar` 解压 jar 包，看看 jar 包内容，解压后发现多了一个 `META-INF/MANIFEST.MF` 文件，内容如下:

```
Manifest-Version: 1.0
Created-By: 1.8.0_181 (Oracle Corporation)
Main-Class: com.demo.Hello

```

正是因为在 `MANIFEST.MF` 文件中指定了 `Main-Class`, jar 才可运行，当然也可在运行时指定 main class: `java -cp hello.jar com.demo.Hello`。有兴趣的话，可以看看 SpringBoot 打出来的 jar 包 Main-Class 是什么，这里贴个例子:

```
Manifest-Version: 1.0
Implementation-Title: demo
Implementation-Version: 0.0.1-SNAPSHOT
Start-Class: com.demo.HelloApplication
Spring-Boot-Classes: BOOT-INF/classes/
Spring-Boot-Lib: BOOT-INF/lib/
Build-Jdk-Spec: 1.8
Spring-Boot-Version: 2.1.6.RELEASE
Created-By: Maven Archiver 3.4.0
Main-Class: org.springframework.boot.loader.JarLauncher

```

jar 包里除了 `MANIFEST.MF` 与 我们定义的 class 文件外，并没有 java 标准库里的 class(eg: Map, String etc), 那么 jar 在运行时是如何感知标准库的存在呢？这里就涉及了 java class 加载流程。

## Java jar 加载 & 执行

当我们执行 `java -jar hello.jar` 时，会启动一个 jvm, 并加载相应的 class 文件至内存，负责加载 class 文件的称为 class loader, java 里包含三类 class loader:

- Bootstrap class loader
- Extensions class loader
- System class loader

加载顺序为: `system class loader -> extensions class loader -> bootstrap loader`，首先 system class loader 会委托 extensions class loader 加载，extensions class loader 找不到继续向上，让 bootstrap class loader 加载，若 bootstrap class loader 找不到则 system class loader 加载，找不到就会报: `class not found`。

bootstrap class loader 查找路径为: `$JAVA_HOME/jre/lib`, extensions class loader 查找路径为: `$JAVA_HOME/jre/lib/ext/`, system class loader 查找路径为: `$CLASSPATH`(若未指定，默认为当前目录)。classpath可通过 env 指定，也可运行时显式指定。
