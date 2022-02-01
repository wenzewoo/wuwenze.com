+++
title = "在SpringBoot中优雅的集成Dubbo"
date = "2018-10-11 07:05:00"
url = "archives/399"
tags = ["Spring"]
categories = ["后端"]
+++

在springboot中集成dubbo示例(非注解)，废话少说，直入正题。

### pom ###

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter</artifactId>
</dependency>
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-web</artifactId>
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
  <groupId>com.alibaba</groupId>
  <artifactId>dubbo</artifactId>
  <version>2.8.4</version>
  <exclusions>
    <exclusion>
      <artifactId>spring</artifactId>
      <groupId>org.springframework</groupId>
    </exclusion>
  </exclusions>
</dependency>

<dependency>
  <groupId>org.apache.zookeeper</groupId>
  <artifactId>zookeeper</artifactId>
  <version>3.4.6</version>
  <exclusions>
    <exclusion>
      <groupId>org.slf4j</groupId>
      <artifactId>slf4j-log4j12</artifactId>
    </exclusion>
    <exclusion>
      <groupId>log4j</groupId>
      <artifactId>log4j</artifactId>
    </exclusion>
  </exclusions>
</dependency>

<dependency>
  <groupId>com.github.sgroschupf</groupId>
  <artifactId>zkclient</artifactId>
  <version>0.1</version>
</dependency>
```

### springboot-dubbo-example-provider ###


```yaml
server:
  port: 10051

dubbo:
  registry:
    address: localhost:2181
```

dubbo-provider.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:dubbo="http://code.alibabatech.com/schema/dubbo"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
       http://code.alibabatech.com/schema/dubbo http://code.alibabatech.com/schema/dubbo/dubbo.xsd">
  <dubbo:application name="springboot-dubbo-example-provider"/>
  <dubbo:registry id="zookeeper" protocol="zookeeper" address="${dubbo.registry.address}"/>
  <dubbo:protocol name="dubbo" port="30001"/>
  <dubbo:service interface="com.wuwenze.springbootdubboexampleprovider.service.SayHelloService"
                 ref="sayHelloService"
                 registry="zookeeper"/>
  <bean id="sayHelloService" class="com.wuwenze.springbootdubboexampleprovider.service.impl.SayHelloServiceImpl"/>
</beans>
```


```java
package com.wuwenze.springbootdubboexampleprovider.service.impl;

import com.wuwenze.springbootdubboexampleprovider.service.SayHelloService;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;

/**
 * @author wwz
 * @version 1 (2018/10/10)
 * @since Java7
 */
@Slf4j
@Service
public class SayHelloServiceImpl implements SayHelloService {
    @Override
    public String sayHello(String name) {
        log.info("call SayHelloService.sayHell(name:{})", name);
        return String.format("hello, %s", name);
    }
}
```


```java
package com.wuwenze.springbootdubboexampleprovider;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.annotation.ImportResource;

@SpringBootApplication
@ImportResource({"classpath:dubbo-provider.xml"})
public class SpringbootDubboExampleProviderApplication {

    public static void main(String[] args) {
        SpringApplication.run(SpringbootDubboExampleProviderApplication.class, args);
        // 若没有引入spring-boot-starter-web依赖，则需要执行System.in.read();让其阻塞。
    }
}
```

### springboot-dubbo-example-consumer ###

```yaml
server:
  port: 10051

dubbo:
  registry:
    address: localhost:2181
```

dubbo-consumer.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:dubbo="http://code.alibabatech.com/schema/dubbo"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
       http://code.alibabatech.com/schema/dubbo http://code.alibabatech.com/schema/dubbo/dubbo.xsd">
  <dubbo:application name="springboot-dubbo-example-consumer"/>

  <dubbo:registry id="zookeeper" protocol="zookeeper" address="${dubbo.registry.address}"/>

  <dubbo:reference id="sayHelloService" interface="com.wuwenze.springbootdubboexampleconsumer.service.SayHelloService"
                   registry="zookeeper" protocol="dubbo" timeout="15000"/>
</beans>
```


```java
package com.wuwenze.springbootdubboexampleconsumer;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.annotation.ImportResource;

@SpringBootApplication
@ImportResource({"classpath:dubbo-consumer.xml"})
public class SpringbootDubboExampleConsumerApplication {

    public static void main(String[] args) {
        SpringApplication.run(SpringbootDubboExampleConsumerApplication.class, args);
        // 若没有引入spring-boot-starter-web依赖，则需要执行System.in.read();让其阻塞。
    }
}
```

```java
package com.wuwenze.springbootdubboexampleconsumer;

import com.wuwenze.springbootdubboexampleconsumer.service.SayHelloService;
import lombok.extern.slf4j.Slf4j;
import org.junit.Assert;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.junit4.SpringRunner;

import javax.annotation.Resource;

@Slf4j
@RunWith(SpringRunner.class)
@SpringBootTest(classes = SpringbootDubboExampleConsumerApplication.class)
public class SpringbootDubboExampleConsumerApplicationTests {

    @Resource
    private SayHelloService sayHelloService;

    @Test
    public void testSayHello() {
        String sayHello = sayHelloService.sayHello("world");
        log.info("test sayHelloService.sayHello() = {}", sayHello);
        Assert.assertEquals(sayHello, "hello, world");
    }
}
```

![f952b713-22d9-4cd5-8e11-befc8a7037a3.png][]


[f952b713-22d9-4cd5-8e11-befc8a7037a3.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20181011/f952b713-22d9-4cd5-8e11-befc8a7037a3.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg