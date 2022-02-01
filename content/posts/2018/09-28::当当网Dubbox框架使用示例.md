+++
title = "当当网Dubbox框架使用示例"
date = "2018-09-28 05:59:00"
url = "archives/89"
tags = ["Dubbo"]
categories = ["后端"]
+++

`Dubbox`是当当网基于阿里巴巴`dubbo`衍生出来的一个新版本，以下是在官网摘抄的相关特性：  

 *  支持REST风格远程调用（HTTP + JSON/XML)
 *  支持基于Jackson的JSON序列化
 *  支持基于嵌入式Tomcat的HTTP remoting体系
 *  升级ZooKeeper客户端：将dubbo中的zookeeper客户端升级到最新的版本，以修正老版本中包含的bug。
 *  支持完全基于Java代码的Dubbo配置：基于Spring的Java Config，实现完全无XML的纯Java代码方式来配置dubbo

### Dubbox 环境搭建 ###

> 由于dubbox相关的jar没有发布到maven中心仓库，故只能自行下载源码编译然后打入本地仓库或私服使用。

以下编译过程均在windows环境上执行，使用git bash.exe

```bash
$ git clone https://github.com/dangdangdotcom/dubbox.git
$ cd dubbox
$ mvn install -Dmaven.test.skip=true
```

### Zookeeper 环境搭建 ###

> 默认使用zookeeper作为服务注册中心，故需要先行安装（步骤略）

修改配置文件zoo.cfg：

```bash
tickTime=2000
initLimit=10
syncLimit=5
dataDir=D:\zookeeper-3.4.11\data
dataLogDir=D:\zookeeper-3.4.11\log
clientPort=2181
```

启动服务：`D:zookeeper-3.4.11binzkServer.cmd`  
使用ZooInspector工具查看相关的数据：

![7820422a-234c-4558-afdb-7d0f388c7e70.png][]

### Dubbox-demo Preview ###

**1) 在IDEA中导入dubbox提供的示例项目：dubboxdubbo-demo**

![9b1cac4c-6f92-4cfd-bd94-2298d5241c81.png][]

**2) 运行服务提供者（`com.alibaba.dubbo.demo.provider.DemoProvider`）**

**3) 运行成功后，在ZooInspector中可以看到相关的信息已经注册上来了。**

![4c9e3abc-4f15-42b9-9f89-f4dd6d3f8952.png][]

**4) 运行服务消费者（`com.alibaba.dubbo.demo.consumer.DemoConsumer`）**

**5) 服务消费着调用测试**

![5f5a2bc3-6875-4c6a-bb72-6979fc4ef9ae.png][]

**6) 运行对外提供的RESTFul接口（com.alibaba.dubbo.demo.consumer.RestClient）**

**7) 使用浏览器访问`http://localhost:8888/services/users/100.json`：**

![6b9c88c2-6b32-4385-a8ec-f68b062d76c1.png][]

**8) 同时支持序列化为xml结果：`http://localhost:8888/services/users/100.xml`：**

![6ac6769b-fe60-4ba5-802a-b586902793ac.png][]

### Dubbox-example ###

**UserRestService.java**

```java
package com.alibaba.dubbo.demo.user.facade;

import com.alibaba.dubbo.demo.user.User;

import javax.validation.constraints.Min;

/**
 * This interface acts as some kind of service broker for the original UserService

 * Here we want to simulate the twitter/weibo rest api, e.g.
 *
 * http://localhost:8888/user/1.json
 * http://localhost:8888/user/1.xml
 *
 * @author lishen
 */
public interface UserRestService {

    /**
     * the request object is just used to test jax-rs injection.
     */
    User getUser(@Min(value=1L, message="User ID must be greater than 1") Long id);

    RegistrationResult registerUser(User user);
}
```

**UserRestServiceImpl.java**

```java
package com.alibaba.dubbo.demo.user.facade;

import com.alibaba.dubbo.demo.user.User;
import com.alibaba.dubbo.demo.user.UserService;
import com.alibaba.dubbo.rpc.RpcContext;
import com.alibaba.dubbo.rpc.protocol.rest.support.ContentType;

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import javax.ws.rs.Consumes;
import javax.ws.rs.GET;
import javax.ws.rs.POST;
import javax.ws.rs.Path;
import javax.ws.rs.PathParam;
import javax.ws.rs.Produces;
import javax.ws.rs.core.MediaType;

@Path("users")
@Consumes({MediaType.APPLICATION_JSON, MediaType.TEXT_XML})
@Produces({ContentType.APPLICATION_JSON_UTF_8, ContentType.TEXT_XML_UTF_8})
public class UserRestServiceImpl implements UserRestService {
    private UserService userService;

    public void setUserService(UserService userService) {
        this.userService = userService;
    }

    @GET
    @Path("{id : \d+}")
    public User getUser(@PathParam("id") Long id) {
        if (RpcContext.getContext().getRequest(HttpServletRequest.class) != null) {
            System.out.println("Client IP address from RpcContext: " + RpcContext.getContext().getRequest(HttpServletRequest.class).getRemoteAddr());
        }
        if (RpcContext.getContext().getResponse(HttpServletResponse.class) != null) {
            System.out.println("Response object from RpcContext: " + RpcContext.getContext().getResponse(HttpServletResponse.class));
        }
        return userService.getUser(id);
    }

    @POST
    @Path("register")
    public RegistrationResult registerUser(User user) {
        return new RegistrationResult(userService.registerUser(user));
    }
}
```


[7820422a-234c-4558-afdb-7d0f388c7e70.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20180928/7820422a-234c-4558-afdb-7d0f388c7e70.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[9b1cac4c-6f92-4cfd-bd94-2298d5241c81.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20180928/9b1cac4c-6f92-4cfd-bd94-2298d5241c81.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[4c9e3abc-4f15-42b9-9f89-f4dd6d3f8952.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20180928/4c9e3abc-4f15-42b9-9f89-f4dd6d3f8952.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[5f5a2bc3-6875-4c6a-bb72-6979fc4ef9ae.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20180928/5f5a2bc3-6875-4c6a-bb72-6979fc4ef9ae.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[6b9c88c2-6b32-4385-a8ec-f68b062d76c1.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20180928/6b9c88c2-6b32-4385-a8ec-f68b062d76c1.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[6ac6769b-fe60-4ba5-802a-b586902793ac.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20180928/6ac6769b-fe60-4ba5-802a-b586902793ac.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg