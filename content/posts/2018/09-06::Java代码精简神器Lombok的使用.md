+++
title = "Java代码精简神器Lombok的使用"
date = "2018-09-06 06:52:00"
url = "archives/173"
tags = ["Java","Lombok"]
categories = ["后端"]
+++

Java 代码中有很多冗余、臃肿的代码（如：Getter/Setter、构造方法、hashCode 方法等）`lombok` 是一款 IDE 插件，其专注于消除此类代码，以达到代码简洁高效的目的。它同时提供了 IDEA 以及 Eclipse 相关的插件，本文以 IDEA 为例，做一些相关的示例

### 准备工作 ###

1）IDEA 中安装相关的插件，如图：

![ef4450ef-8d93-481b-97d2-f1fff9862db7.png][]

2）IDEA 编译器相关设置，如图：

![25a86de8-144e-4ea0-ba99-b99e152059bc.png][]

### 构建项目 ###

以 SpringBoot 项目为例，以下为 pom.xml 文件：

```xml
<dependency>
  <groupId>org.projectlombok</groupId>
  <artifactId>lombok</artifactId>
  <optional>true</optional>
</dependency>
```

### 常用注解 ###

lombok 提供了很多注解，常用的如下。

| 注解                 | 作用                                                                                 |
|--------------------|------------------------------------------------------------------------------------|
| Getter             | 注解在属性上，提供 getter 方法；                                                               |
| Setter             | 注解在属性上，提供 setter 方法；                                                               |
| AllArgsConstructor | 注解在类上，提供构造方法，参数为所有属性；                                                              |
| NoArgsConstructor  | 注解在类上，提供无参构造方法；                                                                    |
| Data               | 注解在类上，提供所有属性的 getter 方法、setter 方法以及 equals、hashCode、toString 等方法；                  |
| Log                | 注解在类上，提供一个名为 log 的属性，类型为 java.util.logging.Logger，也可使用 @Log4j、@Log4j2、Slf4j 等其他注解； |
| ToString           | 注解在类上，提供 toString 方法；                                                              |
| EqualsAndHashCode  | 注解在类上，提供 equals、hashCode 方法；                                                       |
| Synchronized       | 注解在方法上，提供 synchronized，可以指定锁的名称；                                                   |
| NonNull            | 注解在方法参数上，提供对参数的校验，防止空指针异常；                                                         |
| Cleanup            | 注解在局部变量上，提供对资源的关闭，即调用 close 方法；                                                    |


### 使用示例 ###

```java
package com.example.lombok;

import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;
import lombok.ToString;
import lombok.experimental.Accessors;
import lombok.extern.slf4j.Slf4j;

@Data
@ToString
@Accessors(chain = true)
@NoArgsConstructor
@AllArgsConstructor
public class User {
    private Integer uid;
    private String username;
    private String password;
}

@Slf4j
class UserService {

    public static void main(String[] args) {
        User user = new User().setUid(1).setUsername("username").setPassword("123");
        log.debug("user = {}", user);
    }
}
```

直接运行看下结果是否正确吧：

```bash
21:26:31.742 [main] DEBUG com.example.lombok.UserService - user = User(uid=1, username=username, password=123)
```

如此一来，代码变得相当的简洁明了。

### 最终生成的代码结构 ###

lombok 通过插件在编译期间根据注解自动生成了相关的代码，通过下面的代码结构图可以看出一些端倪：

![0929586c-0e0b-408a-9ac6-7f6f0107f20c.png][]


[ef4450ef-8d93-481b-97d2-f1fff9862db7.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20180906/ef4450ef-8d93-481b-97d2-f1fff9862db7.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[25a86de8-144e-4ea0-ba99-b99e152059bc.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20180906/25a86de8-144e-4ea0-ba99-b99e152059bc.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[0929586c-0e0b-408a-9ac6-7f6f0107f20c.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20180906/0929586c-0e0b-408a-9ac6-7f6f0107f20c.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg