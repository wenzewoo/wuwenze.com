+++
title = "Spring Cloud Config 高可用架构"
date = "2018-03-31 07:18:00"
url = "archives/414"
tags = ["Spring","Java","SpringCloud"]
categories = ["后端"]
+++

### 何为高可用? ###

高可用HA（***High Availability***）是分布式系统架构设计中必须考虑的因素之一,它通常是指,通过设计减少系统不能提供服务的时间.

1.  假设系统一直能够提供服务,我们说系统的可用性是100%.
2.  如果系统每运行100个时间单位,会有1个时间单位无法提供服务,我们说系统的可用性是99%.

举个例子,百度的搜索首页是业界公认的高可用保障非常出色的系统

我们通常会通过ping baidu.com来判断网络是否通畅,这也恰巧说明了百度首页的可用性非常之高,值得信赖.

### 高可用的实现方式 ###

1.  ***主从复制:*** 主服务挂掉后,从服务升级为主服务继续工作.
2.  ***双机热备:*** 一台工作,一台备用,工作服务器挂掉后,备用服务器继续工作.
3.  ***分布式集群:*** 多台实例同时工作,当其中一台挂掉后,前端或者代理踢出这台服务器,负载均衡算法将不再调度它.

### Config Server高可用的实现 ###

Config Server 的高可用方案,是借助Eureka(注册中心)实现的,也就是上面提到的\*\**分布式集群*\*\*方案.

多个Config Server同时工作,任何一台挂掉后,Eureka服务器都会通知客户端, 客户端后续将不再从这里请求配置信息.

![9e01764d-176a-4f4c-9aaf-c7054a8c6eed.png][]

### 1. 将Config Server注册到Eureka ###

#### pom.xml ####

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-eureka</artifactId>
</dependency>
```

#### application.yml ####

```yaml
### 将配置中心注册到eureka实现高可用
eureka:
  instance:
    hostname: fastjee-config.com
    ### 更改eureka更新频率, 关闭eureka的自我保护机制.
    leaseRenewalIntervalInSeconds: 10 ### 租期更新时间间隔(默认30秒)
    leaseExpirationDurationInSeconds: 30 ### 租期到期时间(默认90秒)
  client:
    serviceUrl:
      defaultZone: https://fastjee:123456@fastjee-registration.com:5000/eureka/
    healthcheck:
      enabled: true ### 开启健康检查（需要spring-boot-starter-actuator依赖）
```

> 这里更改了eureka的自我保护机制, 为了方便后面的测试, 即时剔除无效服务器.

#### ConfigApplication.java ####

添加@EnableDiscoveryClient注解, 标识该服务为eureka客户端.

```java
@EnableConfigServer
@EnableDiscoveryClient
@SpringBootApplication(scanBasePackages = {"com.fastjee.common.web","com.fastjee.config"})
public class ConfigApplication {

    public static void main(String[] args) {
        SpringApplication.run(ConfigApplication.class, args);
    }
}
```

> @EnableDiscoveryClient可以替换为@EnableEurekaClient,但后者使用场景比较单一,不兼容其它类型的注册中心.

### 2. 测试 ###

在测试前, 建立测试用的Config Clinet项目;

#### 添加配置文件 ####

> 将配置文件fastjee-config-server-test-dev.yml push到由configServer指定的github仓库.

```yaml
config-test:
  sayHello: HelloWorld!
```

#### 搭建测试客户端 ####

pom.xml

```xml
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-config</artifactId>
</dependency>
```

bootstrap.yml

```yaml
spring:
  profiles:
    active: dev
  application:
    name: fastjee-config-server-test
  ### 从配置中心拉取配置, 其他配置文件都在git仓库上
  cloud:
    config:
      fail-fast: true
      discovery:
        service-id: fastjee-config
        enabled: true
      label: master
      profile: ${spring.profiles.active}
      name: ${spring.application.name}

### 注册到注册中心
eureka:
  instance:
    hostname: fastjee-config-server-test.com
  client:
    serviceUrl:
      defaultZone: https://fastjee:123456@fastjee-registration.com:5000/eureka/
```

测试RESTFul接口.

```java
@Value("${config-test.sayHello}")
private String sayHello;

@GetMapping("/sayHello")
public @ResponseBody ResponseEntity<String> sayHello() {

    return ResponseEntity.ok(sayHello);
}
```

#### 开始测试 ####

##### 启动相关服务: #####

1.  Eureka Server
2.  Config Server (同时启动两个 => port: 5001, port: 5009)
3.  测试客户端

启动后的效果图如下:

![4f745c8b-4e28-4982-913c-803c7495df11.png][]

##### 测试步骤: #####

1.  当configServer:5001,configServer:5009同时正常工作时,测试客户端可以启动并拉取配置.
2.  多次启动测试客户端,观察拉取配置的服务器是否在5001/5009之间负载均衡.
3.  将configServer:5009实例下线, 测试客户端依然可以正常启动,且接口访问正常.
4.  将configServer全部下线, 测试客户端启动失败.


[9e01764d-176a-4f4c-9aaf-c7054a8c6eed.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20180331/9e01764d-176a-4f4c-9aaf-c7054a8c6eed.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[4f745c8b-4e28-4982-913c-803c7495df11.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20180331/4f745c8b-4e28-4982-913c-803c7495df11.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg