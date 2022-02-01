+++
title = "Spring5响应式WEB编程-Webflux示例"
date = "2018-12-17 06:38:00"
url = "archives/378"
tags = ["Spring","Webflux"]
categories = ["后端"]
+++

### Spring WebFlux简介 ###

`Spring WebFlux`是随`Spring 5`推出的响应式Web框架：

![6f47b4d4-8333-42c5-800c-5be2e7cc478a.png][]

（左侧为基于`spring-webmvc`的技术栈，右侧为基于`spring-webflux`的技术栈）

#### 服务端技术栈 ####

 *  `Spring WebFlux`是基于响应式流的，因此可以用来建立异步的、非阻塞的、事件驱动的服务。它采用Reactor作为首选的响应式流的实现库，不过也提供了对`RxJava`的支持；
 *  由于响应式编程的特性，Spring WebFlux和Reactor底层需要支持异步的运行环境，比如`Netty`和`Undertow`；也可以运行在支持异步I/O的Servlet 3.1的容器之上，比如Tomcat（8.0.23及以上）和Jetty（9.0.4及以上）；
 *  从图的纵向上看，spring-webflux上层支持两种开发模式：
    
     *  类似于Spring WebMVC的基于注解（`@Controller`、`@RequestMapping`）的开发模式；
     *  Java 8 `lambda` 风格的函数式开发模式；
 *  响应式的`WebSocket`服务端开发；

#### 客户端技术栈 ####

此外，Spring WebFlux也提供了一个响应式的Http客户端API `WebClient`。它可以用函数式的方式异步非阻塞地发起Http请求并处理响应。

WebClient可以看做是响应式的`RestTemplate`，相对于后者来说，他的特点是：

 *  是非阻塞的，可以基于少量的线程处理更高的并发；
 *  可以使用Java 8 `lambda`表达式；
 *  支持异步的同时也可以支持同步的使用方式；
 *  可以通过数据流的方式与服务端进行双向通信；
 *  响应式的`WebSocket`客户端API开发；

### 通过Spring Boot实战WebFlux ###

本文的例子很简单，先直接使用`Spring Initializr`构建一个简单的`SpringBoot`项目。

#### 核心依赖 ####

![043e56a0-1170-4e5c-b242-89ab5fbff20b.png][]

对应的完整POM文件如下：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.1.1.RELEASE</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>
    <groupId>com.wuwenze</groupId>
    <artifactId>spring-webflux</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>spring-webflux</name>
    <description>Demo project for Spring Boot</description>

    <properties>
        <java.version>1.8</java.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-webflux</artifactId>
        </dependency>

        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>io.projectreactor</groupId>
            <artifactId>reactor-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>

</project>
```

#### Hello WebFlux ####

![a2640f0f-3802-4f7d-a30f-c484cd1037ae.png][]

从语法上来看，与传统的SpringMVC看起来还是相差无几，启动应用后，发现应用启动于`Netty`之上。

WebFlux提供了与之前WebMVC相同的一套注解来定义请求的处理，使得Spring使用者迁移到响应式开发方式的过程变得异常轻松。

整个技术栈从命令式的、同步阻塞的【`Spring-WebMVC + Servlet + Tomcat`】变成了响应式的、异步非阻塞的【`Spring-WebFlux + Reactor + Netty`】。

#### WebFlux 函数式开发模式 ####

这里还是先来个简单的例子吧，后续再详细讲解：

UserService

```java
package com.wuwenze.springwebflux.service;

import com.google.common.collect.Maps;
import com.wuwenze.springwebflux.entity.User;
import org.springframework.stereotype.Service;
import reactor.core.publisher.Flux;
import reactor.core.publisher.Mono;

import java.util.Map;

@Service
public class UserService {
    // key, user
    private final Map<String, User> _userMock = Maps.newConcurrentMap();

    {
        _userMock.put("key1", User.builder().id(1).name("user1").build());
        _userMock.put("key2", User.builder().id(2).name("user2").build());
        _userMock.put("key3", User.builder().id(3).name("user3").build());
        _userMock.put("key4", User.builder().id(4).name("user4").build());
        _userMock.put("key5", User.builder().id(5).name("user5").build());
    }

    public Flux<User> list() {
        return Flux.fromIterable(this._userMock.values());
    }

    public Flux<User> findByKeys(Flux<String> keys) {
        return keys.flatMap((key) -> Mono.justOrEmpty(this._userMock.get(key)));
    }

    public Mono<User> getByKey(String key) {
        return Mono.justOrEmpty(this._userMock.get(key))
                .switchIfEmpty(Mono.error(new RuntimeException("#key = " + key + "不存在")));
    }

    public Mono<User> saveOrUpdate(User user) {
        String key = "key" + user.getId();
        return this.save(key, user)
                .onErrorResume((e) -> {
                    // 如果存在，说明数据库存在记录，查找并修改
                    return this.getByKey(key)
                            .flatMap((originalUser) -> {
                                originalUser.setName(user.getName());
                                return this.update(key, originalUser);
                            });
                });
    }

    public Flux<User> saveOrUpdateBatch(Flux<User> users) {
        return users.doOnNext((user) -> this.saveOrUpdate(user));
    }

    public Mono<User> remove(String key) {
        return Mono.justOrEmpty(this._userMock.remove(key));
    }

    // 模拟数据库新增
    private Mono<User> save(String key, User user) {
        if (this._userMock.containsKey(key)) {
            return Mono.error(new RuntimeException());
        }
        this._userMock.put(key, user);
        return Mono.just(user);
    }

    // 模拟数据库修改
    private Mono<User> update(String key, User user) {
        this._userMock.put(key, user);
        return Mono.just(user);
    }
}
```

UserController

```java
package com.wuwenze.springwebflux.rest;

import com.wuwenze.springwebflux.entity.User;
import com.wuwenze.springwebflux.service.UserService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.MediaType;
import org.springframework.web.bind.annotation.*;
import reactor.core.publisher.Flux;
import reactor.core.publisher.Mono;

import java.time.Duration;

@RestController
public class UserController {

    @Autowired
    public UserService userService;

    @GetMapping("/list")
    public Flux<User> list() {
        return this.userService.list();
    }

    // 流式响应，数据每延迟2秒，一批一批到达客户端
    @GetMapping(value = "/list_stream", produces = MediaType.APPLICATION_STREAM_JSON_VALUE)
    public Flux<User> list_stream() {
        return this.userService.list()
                .delayElements(Duration.ofSeconds(2));
    }

    @GetMapping("/get/{key}")
    public Mono<User> get(@PathVariable String key) {
        return this.userService.getByKey(key);
    }

    @PostMapping("/save")
    public Mono<User> save(@RequestBody User user) {
        return this.userService.saveOrUpdate(user);
    }

    @DeleteMapping("/remove/{key}")
    public Mono<User> remove(@PathVariable String key) {
        return this.userService.remove(key);
    }
}
```


[6f47b4d4-8333-42c5-800c-5be2e7cc478a.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20181217/6f47b4d4-8333-42c5-800c-5be2e7cc478a.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[043e56a0-1170-4e5c-b242-89ab5fbff20b.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20181217/043e56a0-1170-4e5c-b242-89ab5fbff20b.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[a2640f0f-3802-4f7d-a30f-c484cd1037ae.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20181217/a2640f0f-3802-4f7d-a30f-c484cd1037ae.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg