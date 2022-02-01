+++
title = "SpringBoot Admin 集成指南(v2.1.1)"
date = "2019-01-11 06:56:00"
url = "archives/386"
tags = ["Spring"]
categories = ["后端","运维"]
+++

SpringBoot Admin用于管理和监控SpringBoot应用程序。 应用程序作为Spring Boot Admin Client向为Spring Boot Admin Server注册（通过HTTP）或使用SpringCloud注册中心（例如Eureka，Consul）发现。

其常见的功能如下：

 *  显示健康状况
 *  显示详细信息，例如
    
     *  JVM和内存指标
     *  micrometer.io指标
     *  数据源指标
     *  缓存指标
 *  显示构建信息编号
 *  关注并下载日志文件
 *  查看jvm系统和环境属性
 *  查看Spring Boot配置属性
 *  支持Spring Cloud的postable / env-和/ refresh-endpoint
 *  轻松的日志级管理
 *  与JMX-beans交互
 *  查看线程转储
 *  查看http跟踪
 *  查看auditevents
 *  查看http-endpoints
 *  查看计划任务
 *  查看和删除活动会话（使用spring-session）
 *  查看Flyway / Liquibase数据库迁移
 *  下载heapdump文件
 *  状态变更通知（通过电子邮件，Slack，Hipchat，......）
 *  状态更改的事件日志（非持久性）

### 开始使用 ###

#### 创建SpringBoot AdminServer ####

建立标准的Maven项目，其完整的依赖如下：

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
    <artifactId>admin-server</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>admin-server</name>
    <description>Demo project for Spring Boot</description>

    <properties>
        <java.version>1.8</java.version>
        <spring-boot-admin.version>2.1.1</spring-boot-admin.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>de.codecentric</groupId>
            <artifactId>spring-boot-admin-starter-server</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>de.codecentric</groupId>
                <artifactId>spring-boot-admin-dependencies</artifactId>
                <version>${spring-boot-admin.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>

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

然后在工程的启动类AdminServerApplication加上@EnableAdminServer注解，开启AdminServer的功能:

```java
package com.wuwenze.adminserver;

import de.codecentric.boot.admin.server.config.EnableAdminServer;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@EnableAdminServer
@SpringBootApplication
public class AdminServerApplication {

    public static void main(String[] args) {
        SpringApplication.run(AdminServerApplication.class, args);
    }

}
```

指定端口和应用名称：

```yaml
server:
  port: 10086
spring:
  application:
    name: admin-server
```

#### 创建SpringBoot Admin Client ####

再创建一个Maven项目，核心依赖如下（省略了部分）：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
<dependency>
    <groupId>de.codecentric</groupId>
    <artifactId>spring-boot-admin-starter-client</artifactId>
</dependency>
```

在配置文件中注册到adminServer中，同时暴露自己的所有actuator端点信息：

```yaml
server:
  port: 10087
spring:
  application:
    name: admin-client
  boot:
    admin:
      client:
        url: http://localhost:10086

management:
  endpoints:
    web:
      exposure:
        include: '*'
  endpoint:
    health:
      show-details: ALWAYS
```

启动类无需做其他设置：

```java
package com.wuwenze.adminclient;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class AdminClientApplication {

    public static void main(String[] args) {
        SpringApplication.run(AdminClientApplication.class, args);
    }

}
```

![58fc2a89-c57b-44bd-b2f0-50f31d8adbdf.png][]

依次启动AdminServerApplication、AdminClientApplication

通过浏览器访问[http://localhost:10086][http_localhost_10086]

![ca1b9617-437b-462e-95ce-a29c26eafac0.png][]

查看内存状态信息：

![16c0a172-1b71-4a61-bf95-b1cbd0a3eda8.png][]

SpringBean相关的情况：

![d30d3bea-4500-48b7-a413-a14036f4c52c.png][]

日志级管理：

![e6fb6c49-eb40-4a6c-9388-0054144325de.png][]

还有更多的监控信息，这里就不再一一列举了，自己研究就好。

### 集成注册中心 ###

使用Eureka搭建一个注册中心（其他的也行，这里不表）

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
    <artifactId>eureka-server</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>eureka-server</name>
    <description>Demo project for Spring Boot</description>

    <properties>
        <java.version>1.8</java.version>
        <spring-cloud.version>Greenwich.RC2</spring-cloud.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>${spring-cloud.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>

    <repositories>
        <repository>
            <id>spring-milestones</id>
            <name>Spring Milestones</name>
            <url>https://repo.spring.io/milestone</url>
        </repository>
    </repositories>

</project>
```

注册中心的相关配置，同样也暴露了所有的端点信息：

```yaml
server:
  port: 8761

spring:
  application:
    name: eureka-server

eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka
    register-with-eureka: false
    fetch-registry: false

management:
  endpoints:
    web:
      exposure:
        include: "*"
  endpoint:
    health:
      show-details: ALWAYS
```

启动类中加上@EnableEurekaServer注解，开启EurekaServer:

```java
package com.wuwenze.euerkaserver;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.netflix.eureka.server.EnableEurekaServer;

@EnableEurekaServer
@SpringBootApplication
public class EuerkaServerApplication {

    public static void main(String[] args) {
        SpringApplication.run(EuerkaServerApplication.class, args);
    }

}
```

改造admin-server，加入eureka客户端相关依赖：

```xml
<properties>
    <java.version>1.8</java.version>
    <spring-boot-admin.version>2.1.1</spring-boot-admin.version>
    <spring-cloud.version>Finchley.SR2</spring-cloud.version>
</properties>

<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>de.codecentric</groupId>
        <artifactId>spring-boot-admin-starter-server</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-security</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>

<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>${spring-cloud.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
        <dependency>
            <groupId>de.codecentric</groupId>
            <artifactId>spring-boot-admin-dependencies</artifactId>
            <version>${spring-boot-admin.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

再将其注册到Eureka注册中心，同时也暴露所有端点信息：

```yaml
server:
  port: 10086
spring:
  application:
    name: admin-server

eureka:
  client:
    registryFetchIntervalSeconds: 5
    service-url:
      defaultZone: ${EUREKA_SERVICE_URL:http://localhost:8761}/eureka/
  instance:
    leaseRenewalIntervalInSeconds: 10
    health-check-url-path: /actuator/health

management:
  endpoints:
    web:
      exposure:
        include: "*"
  endpoint:
    health:
      show-details: ALWAYS
```

同时，在启动类中，通过`@EnableDiscoveryClient`开启EurekaClient功能：

```java
package com.wuwenze.adminserver;

import de.codecentric.boot.admin.server.config.EnableAdminServer;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;

@EnableAdminServer
@EnableDiscoveryClient
@SpringBootApplication
public class AdminServerApplication {

    public static void main(String[] args) {
        SpringApplication.run(AdminServerApplication.class, args);
    }

}
```

同理，改造admin-client，将其也注册到Eureka中心去：

```xml
<properties>
    <java.version>1.8</java.version>
    <spring-boot-admin.version>2.0.4</spring-boot-admin.version>
    <spring-cloud.version>Finchley.SR2</spring-cloud.version>
</properties>

<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>de.codecentric</groupId>
        <artifactId>spring-boot-admin-starter-client</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>

<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>${spring-cloud.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
        <dependency>
            <groupId>de.codecentric</groupId>
            <artifactId>spring-boot-admin-dependencies</artifactId>
            <version>${spring-boot-admin.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

相关的配置文件、启动类：

```yaml
server:
  port: 8762
spring:
  application:
    name: admin-client
eureka:
  instance:
    leaseRenewalIntervalInSeconds: 10
    health-check-url-path: /actuator/health

  client:
    registryFetchIntervalSeconds: 5
    service-url:
      defaultZone: ${EUREKA_SERVICE_URL:http://localhost:8761}/eureka/
management:
  endpoints:
    web:
      exposure:
        include: "*"
  endpoint:
    health:
      show-details: ALWAYS
```

```java
package com.wuwenze.adminclient;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;

@EnableDiscoveryClient
@SpringBootApplication
public class AdminClientApplication {

    public static void main(String[] args) {
        SpringApplication.run(AdminClientApplication.class, args);
    }

}
```

![445123e9-e7c8-408c-8a92-1ff04ff257ca.png][]

如上图所示，依次启动EurekaServerApplication、AdminServerApplication、AdminClientApplication:

![e3feaf49-7d33-4d41-82fc-27cffdaed9f3.png][]

此时可以看到，相关的服务已经注册到Eureka中，并且在SpringBoot Admin中已经上线了。

![c6e4caaf-c68d-404f-9791-12b9153518cb.png][]

### 基于Spring Security的授权验证 ###

默认情况下，SpringBoot Admin处于裸奔的状态，这显然是不太安全的，好在SpringBoot可以很方便的集成Spring Security。

继续改造admin-server工程，在其pom文件中加入相关的依赖：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
```

为了方便，直接在配置文件中写死登陆用户名密码，同时在服务注册时带上相关的metadata-map信息：

```yaml
server:
  port: 10086
spring:
  application:
    name: admin-server
  security:
    user:
      name: "admin"
      password: "admin"

eureka:
  client:
    registryFetchIntervalSeconds: 5
    service-url:
      defaultZone: ${EUREKA_SERVICE_URL:http://localhost:8761}/eureka/
  instance:
    leaseRenewalIntervalInSeconds: 10
    health-check-url-path: /actuator/health
    metadata-map:
      user.name: ${spring.security.user.name}
      user.password: ${spring.security.user.password}

management:
  endpoints:
    web:
      exposure:
        include: "*"
  endpoint:
    health:
      show-details: ALWAYS
```

通过JavaConfig的方式，实现WebSecurityConfigurerAdapter：

```java
package com.wuwenze.adminserver.config;

import de.codecentric.boot.admin.server.config.AdminServerProperties;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.WebSecurityConfigurerAdapter;
import org.springframework.security.web.authentication.SavedRequestAwareAuthenticationSuccessHandler;

@Configuration
public class SecuritySecureConfig extends WebSecurityConfigurerAdapter {
    private String adminContextPath;

    public SecuritySecureConfig(AdminServerProperties adminServerProperties) {
        this.adminContextPath = adminServerProperties.getContextPath();
    }

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        SavedRequestAwareAuthenticationSuccessHandler successHandler = new SavedRequestAwareAuthenticationSuccessHandler();
        successHandler.setTargetUrlParameter("redirectTo");

        http.authorizeRequests()
                .antMatchers(adminContextPath + "/assets/**").permitAll()
                .antMatchers(adminContextPath + "/login").permitAll()
                .anyRequest().authenticated()
                .and()
                .formLogin().loginPage(adminContextPath + "/login").successHandler(successHandler)
                .and()
                .logout().logoutUrl(adminContextPath + "/logout")
                .and()
                .httpBasic()
                .and()
                .csrf().disable();
    }
}
```

重启相关服务，再次访问，发现基本的登陆授权认证页面已经出现了：

![260312a1-f4fd-4bbf-b18a-6b45214d93c5.png][]

### 集成自动邮件预警 ###

作为一个运维监控平台，邮件预警的功能是必不可少的，集成起来也是相当简单，咱们继续改造admin-server:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-mail</artifactId>
</dependency>
```

完善配置文件，加入邮件服务器的相关配置信息：

```yaml
server:
  port: 10086
spring:
  application:
    name: admin-server
  security:
    user:
      name: "admin"
      password: "admin"
  mail:
    host: smtp.163.com
    username: xxxxxxxxxxx
    password: xxxxxxxxxxxx
    properties:
      mail.debug: false  
      mail.smtp.auth: true
      mail.smtp.port: 465
      mail.smtp.ssl.enable: true
      mail.smtp.ssl.socketFactory: sf  
  boot:
    admin:
      notify:
        mail:
          to: 877037201@qq.com
          from: 17311223306@163.com

eureka:
  client:
    registryFetchIntervalSeconds: 5
    service-url:
      defaultZone: ${EUREKA_SERVICE_URL:http://localhost:8761}/eureka/
  instance:
    leaseRenewalIntervalInSeconds: 10
    health-check-url-path: /actuator/health
    metadata-map:
      user.name: ${spring.security.user.name}
      user.password: ${spring.security.user.password}

management:
  endpoints:
    web:
      exposure:
        include: "*"
  endpoint:
    health:
      show-details: ALWAYS
```

完成相关配置后，重启相关服务，然后试着将admin-client手动下线，在配置的收件箱中即会收到相关的邮件预警信息：

![cfabc8f8-18a8-4eef-84dc-0faaa74246d3.png][]

同时，在SpringBoot Admin监控界面中，相关的服务也已经下线了：

![d4626a20-bf7b-4190-ac4f-429d5bf9d559.png][]

有关SpringBoot Admin的相关功能先介绍到这里，还有更多实用的配置就不详解了。


[58fc2a89-c57b-44bd-b2f0-50f31d8adbdf.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20190111/58fc2a89-c57b-44bd-b2f0-50f31d8adbdf.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[http_localhost_10086]: http://localhost:10086/
[ca1b9617-437b-462e-95ce-a29c26eafac0.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20190111/ca1b9617-437b-462e-95ce-a29c26eafac0.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[16c0a172-1b71-4a61-bf95-b1cbd0a3eda8.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20190111/16c0a172-1b71-4a61-bf95-b1cbd0a3eda8.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[d30d3bea-4500-48b7-a413-a14036f4c52c.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20190111/d30d3bea-4500-48b7-a413-a14036f4c52c.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[e6fb6c49-eb40-4a6c-9388-0054144325de.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20190111/e6fb6c49-eb40-4a6c-9388-0054144325de.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[445123e9-e7c8-408c-8a92-1ff04ff257ca.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20190111/445123e9-e7c8-408c-8a92-1ff04ff257ca.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[e3feaf49-7d33-4d41-82fc-27cffdaed9f3.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20190111/e3feaf49-7d33-4d41-82fc-27cffdaed9f3.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[c6e4caaf-c68d-404f-9791-12b9153518cb.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20190111/c6e4caaf-c68d-404f-9791-12b9153518cb.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[260312a1-f4fd-4bbf-b18a-6b45214d93c5.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20190111/260312a1-f4fd-4bbf-b18a-6b45214d93c5.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[cfabc8f8-18a8-4eef-84dc-0faaa74246d3.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20190111/cfabc8f8-18a8-4eef-84dc-0faaa74246d3.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[d4626a20-bf7b-4190-ac4f-429d5bf9d559.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20190111/d4626a20-bf7b-4190-ac4f-429d5bf9d559.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg