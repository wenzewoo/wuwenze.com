+++
title = "使用Consul代替Spring Cloud Eureka"
date = "2018-07-06 07:23:00"
url = "archives/419"
tags = ["Spring","SpringCloud","Consul"]
categories = ["后端"]
+++

随着Eureka 2.0 开源工作宣告停止，其实是可以考虑转战其他方式来实现注册中心了（如：Zookeeper、Redis、Consul等）

本文通过简单的描述，快速将Consul集成到SpringCloud环境中。

##### Consul环境搭建 #####

> 官网：[https://www.consul.io/][https_www.consul.io]

官网提供了（macOS、FreeBSD、Linux、Solaris、Windows）全平台的相关包，下面以Windows为例：

1.  下载 [https://releases.hashicorp.com/consul/1.2.0/consul\_1.2.0\_windows\_amd64.zip][https_releases.hashicorp.com_consul_1.2.0_consul_1.2.0_windows_amd64.zip]
2.  解压，并创建快速启动脚本：

```bash
consul agent -dev
pause
```

![66a0d64a-00c3-4366-bc0e-16d7cbfc5838.png][]

1.  启动，然后浏览器访问：[http://localhost:8500][http_localhost_8500], 出现UI界面则搭建成功。

![459fb5ac-5de3-4a5b-9415-6dd0e9405c58.png][]

##### 将服务注册到Consul #####

###### pom.xml ######

> 这里列举了集成Consul的核心依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency><!--健康检查相关依赖,后面会用到-->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-consul-discovery</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

##### bootstrap.yml #####

```yaml
server:
  port: 8762
spring:
  application:
    name: angercloud-consul
  cloud:
    consul:
      host: localhost
      port: 8500
    discovery:
      client:
        healthCheckInterval: 15s
        instance-id: angercloud-consul
        ### 健康检查URI.由spring-boot-starter-actuator提供。
        healthCheckPath: ${management.contextPath}/actuator/health
```

#### 启动测试 ####

> 通过注解 @EnableDiscoveryClient 标识该服务为发现客户端，然后根据配置文件注册中Consul

```java
package com.angercloud.registry.consul;

import com.angercloud.support.core.consts.AngerCloudPackages;
import com.angercloud.support.core.web.JSONEntity;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

/**
 * @author wwz
 * @version 1 (2018/7/6)
 * @since Java7
 */
@RestController
@EnableDiscoveryClient
@SpringBootApplication(scanBasePackages = {
        AngerCloudPackages.SUPPORT_CORE, AngerCloudPackages.REGISTRY_CONSUL
})
public class AngerCloudConsulApplication {

    public static void main(String[] args) {
        SpringApplication.run(AngerCloudConsulApplication.class, args);
    }

    @GetMapping("/sayHello")
    public JSONEntity<?> sayHello() {
        return JSONEntity.ok("Hello, AngerCloudConsulApplication");
    }
}
```

![d27c2290-7716-4afa-b775-63d75d55531a.png][]![264c2884-9361-412b-8224-08a684549ff4.png][]![3eff4a14-1472-493f-82e5-332d9319d11b.png][]

有关更多细节，后面使用的时候再详细研究（或自己实现redis注册中心）


[https_www.consul.io]: https://www.consul.io/
[https_releases.hashicorp.com_consul_1.2.0_consul_1.2.0_windows_amd64.zip]: https://releases.hashicorp.com/consul/1.2.0/consul_1.2.0_windows_amd64.zip
[66a0d64a-00c3-4366-bc0e-16d7cbfc5838.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20180706/66a0d64a-00c3-4366-bc0e-16d7cbfc5838.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[http_localhost_8500]: http://localhost:8500/
[459fb5ac-5de3-4a5b-9415-6dd0e9405c58.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20180706/459fb5ac-5de3-4a5b-9415-6dd0e9405c58.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[d27c2290-7716-4afa-b775-63d75d55531a.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20180706/d27c2290-7716-4afa-b775-63d75d55531a.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[264c2884-9361-412b-8224-08a684549ff4.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20180706/264c2884-9361-412b-8224-08a684549ff4.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[3eff4a14-1472-493f-82e5-332d9319d11b.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20180706/3eff4a14-1472-493f-82e5-332d9319d11b.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg