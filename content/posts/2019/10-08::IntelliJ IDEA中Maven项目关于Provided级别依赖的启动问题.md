+++
title = "IntelliJ IDEA中Maven项目关于Provided级别依赖的启动问题"
date = "2019-10-08 16:54:00"
url = "archives/656"
tags = ["Intellij IDEA"]
categories = ["开发工具"]
+++

### 概述 ###

在`IntelliJ IDEA`中使用`Maven`项目时，遇到个问题如下：  
![164354\_2185734a\_955503][164354_2185734a_955503]

### 问题分析 ###

看到这个东西，下意识的认为`pom.xml`中缺少了slf4j相关的配置，遂查看之，又是存在的：

```xml
<dependency>
    <groupId>org.slf4j</groupId>
    <artifactId>slf4j-api</artifactId>
    <version>1.7.7</version>
    <scope>provided</scope>
</dependency>
```

有些纳闷，其实项目是能够正常编译的，但是为什么Run不了呢？其实上面贴出来的`<scope>provided</scope>`就已经告诉我们问题了，先看下 scope 相关的定义：

 *  compile:编译依赖范围。如果没有指定，就会默认使用该依赖范围。使用此依赖范围的Maven依赖，对于编译、测试、运行三种classpath都有效。
 *  test: 测试依赖范围，只针对单元测试；
 *  provided:已提供依赖范围。使用此依赖范围的Maven依赖，对于编译和测试classpath有效，但在运行时候无效。

很明显的问题了，IDEA会严格遵守Maven的约定，在运行时不会加载provided级别的包，以前使用Eclipse时是可以直接运行的（Eclipse默认加载，方便了开发者，但是违反了Maven规范，这是好事还是坏事呢？我们就不深究了，反正以后也不会用了，哈哈）

### 解决方案 ###

 *  调整 scope 级别为\[`compile`\] (不推荐，这个要斟酌一下，会导致依赖传递问题，如果是刻意设置的provided，那就得不偿失了)
 *  调整 IDEA \[`Run/Debug Configurations`\] **Include dependencies with ‘Provided’ scope** ：

![170159\_a47692de\_955503][170159_a47692de_955503]  
如此就能正常运行了，可见IDEA还是非常人性化的。


[164354_2185734a_955503]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20191008/72a8f652-1b56-4ac4-9164-d609239311e9.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[170159_a47692de_955503]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20191008/5045bd8f-3d04-4345-b62a-0519199fcc75.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg