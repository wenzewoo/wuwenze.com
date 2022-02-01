+++
title = "开源Java诊断工具Arthas推荐"
date = "2019-07-26 16:45:00"
url = "archives/637"
tags = ["Java"]
categories = ["运维"]
+++

Arthas 是Alibaba开源的Java诊断工具，初步试用了一下，甚是方便，在生产线上找问题应该比较方便。

Arthas可以帮助你解决什么问题？

1.  这个类从哪个 jar 包加载的？为什么会报各种类相关的 Exception？
2.  我改的代码为什么没有执行到？难道是我没 commit？分支搞错了？
3.  遇到问题无法在线上 debug，难道只能通过加日志再重新发布吗？
4.  线上遇到某个用户的数据处理有问题，但线上同样无法 debug，线下无法重现！
5.  是否有一个全局视角来查看系统的运行状况？
6.  有什么办法可以监控到JVM的实时运行状态？

Arthas支持JDK 6+，支持Linux/Mac/Winodws，采用命令行交互模式，同时提供丰富的 Tab 自动补全功能，进一步方便进行问题的定位和诊断。

### 快速安装 ###

远程到生产的Linux主机上，一行代码即可下载工具包。

```bash
wget https://alibaba.github.io/arthas/arthas-boot.jar
```

### 启动 ###

```bash
java -jar arthas-boot.jar
```

![223523\_49eb1786\_955503][223523_49eb1786_955503]

执行命令后，Arthas会列出当前机器上的Java进程，输入对应的编号进入交互式控制台，这时候就可以通过各种命令的组合来对这个Java进程进行诊断了。

同时，也提供了一个Web Console，地址为：[http://127.0.0.1:8563/][http_127.0.0.1_8563] ![223523\_946e6f5e\_955503][223523_946e6f5e_955503]  
但是实际意义估计不大，进去后也是个交互式控制台，还不如直接在终端里操作了。

### dashboard ###

输入[dashboard][]，按回车/enter，会展示当前进程的信息，通过 `Ctrl + C` 可以退出

![223523\_e89912e5\_955503][223523_e89912e5_955503]

### thread ###

可以通过thread命令来打印指定线程的堆栈信息

![223524\_3364fae6\_955503][223524_3364fae6_955503]

```bash
$ thread 96
"DubboSaveRegistryCache-thread-1" Id=96 WAITING on java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject@48ff8704
    at sun.misc.Unsafe.park(Native Method)
    -  waiting on java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject@48ff8704
    at java.util.concurrent.locks.LockSupport.park(LockSupport.java:186)
    at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.await(AbstractQueuedSynchronizer.java:2043)
    at java.util.concurrent.LinkedBlockingQueue.take(LinkedBlockingQueue.java:442)
    at java.util.concurrent.ThreadPoolExecutor.getTask(ThreadPoolExecutor.java:1068)
    at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1130)
    at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:615)
    at java.lang.Thread.run(Thread.java:745)

Affect(row-cnt:0) cost in 154 ms.
```

该命令支持管道，配合grep命令可以筛选关键的一些信息，如：

```bash
$ thread 96 | grep 'runWorker('
    at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1130)
```

### jad ###

使用jad来反编译某个类，这个功能很棒，再也不用下载jar包下来再反编译了。

![223524\_a4f8523f\_955503][223524_a4f8523f_955503]

```bash
jad com.ewei.open.dubbo.http.api.account.OpenRoleApi

ClassLoader:                                                                   
+-WebappClassLoader                                                            
    context:                                                                   
    delegate: false                                                            
    repositories:                                                              
      /WEB-INF/classes/                                                        
  ----------> Parent Classloader:                                              
  java.net.URLClassLoader@6ec5b763                                             
                                                                               
  +-java.net.URLClassLoader@6ec5b763                                           
    +-sun.misc.Launcher$AppClassLoader@305f387c                                
      +-sun.misc.Launcher$ExtClassLoader@3b756db3                              

Location:                                                                      
/Users/wuwenze/Development/ewei-projects/helpdesk/ewei-open/target/ewei-open/WE
B-INF/lib/ewei-module-account-web-api-0.0.1-SNAPSHOT.jar                       

/*
 * Decompiled with CFR 0_132.
 * 
 * Could not load the following classes:
 *  com.ewei.account.api.entity.Role
 *  io.swagger.annotations.Api
 *  io.swagger.annotations.ApiOperation
 *  javax.validation.constraints.NotNull
 *  org.budo.dubbo.protocol.http.authentication.AuthenticationCheck
 */
package com.ewei.open.dubbo.http.api.account;

import com.ewei.account.api.entity.Role;
import io.swagger.annotations.Api;
import io.swagger.annotations.ApiOperation;
import java.util.List;
import javax.validation.constraints.NotNull;
import org.budo.dubbo.protocol.http.authentication.AuthenticationCheck;

@Api(value="u89d2u8272u76f8u5173u63a5u53e3")
public interface OpenRoleApi {
....
```

### watch ###

通过该命令，实时监控某个函数的返回值，这个相当好使了，告别打印日志吧

```bash
$ watch com.ewei.account.api.service.IUserService findByEmailOrMobilePhoneOrExternalId returnobj
Press Q or Ctrl+C to abort.
Affect(class-cnt:2 , method-cnt:2) cost in 162 ms.
ts=2019-07-26 13:01:51; [cost=13.695ms] result=@User[
    serialVersionUID=@Long[6552716967614882077],
    TYPE_ENGINEER=@String[engineer],
    TYPE_CUSTOMER=@String[customer],
    TYPE_SUSPENDED_CUSTOMER=@String[suspended_customer],
    STATUS_NO_VALIDATION=@Integer[2],
    STATUS_NORMAL=@Integer[1],
    STATUS_DISABLE=@Integer[0],
    USER_STATUS_LABELS=@HashMap[isEmpty=false;size=3],
    customerCustomFields=@HashSet[isEmpty=true;size=0],
    engineerRole=null,
    defaultServiceDesk=null,
    token=null,
    browserOpenId=null,
    serviceDesk=null,
    tags=@HashSet[isEmpty=true;size=0],
    userBadgeConfig=null,
]
```

耗时，返回值一目了然，甚是方便，只要系统产生调用，立马在控制台就能监控到。

### trace ###

查看方法的内部调用路径,并返回每个节点的耗时情况.  
![223524\_97f8fdd0\_955503][223524_97f8fdd0_955503]

```bash
$ trace com.ewei.open.dubbo.http.api.impl.account.OpenAccountApiImpl getUserToken
Press Q or Ctrl+C to abort.
Affect(class-cnt:2 , method-cnt:2) cost in 121 ms.
`---ts=2019-07-26 13:42:11;thread_name=http-bio-8280-exec-5;id=1f;is_daemon=true;priority=5;TCCL=org.apache.catalina.loader.WebappClassLoader@27ad510b
    `---[222.084ms] com.ewei.open.dubbo.http.api.impl.account.OpenAccountApiImpl$$EnhancerBySpringCGLIB$$4919e286:getUserToken()
        `---[221.977ms] org.springframework.cglib.proxy.MethodInterceptor:intercept() ##0
            `---[220.917ms] com.ewei.open.dubbo.http.api.impl.account.OpenAccountApiImpl:getUserToken()
                +---[9.928ms] com.ewei.open.dubbo.http.api.context.ApiContext:getCurrentProviderIdNoNull() ##620
                +---[19.447ms] com.ewei.account.api.service.IUserService:findByEmailOrMobilePhoneOrExternalId() ##622
                +---[0.009ms] com.ewei.account.api.entity.Provider:<init>() ##628
                +---[0.472ms] com.ewei.open.dubbo.http.api.context.ApiContext:getRemoteAddr() ##628
                +---[190.131ms] com.ewei.account.logic.ITokenLogic:findOrNewUserToken() ##628
                +---[0.018ms] org.slf4j.Logger:debug() ##629
                +---[0.014ms] com.ewei.account.api.entity.User:getId() ##630
                `---[0.009ms] org.budo.support.lang.util.MapUtil:stringObjectMap() ##630
```

### 退出 ###

如果只是退出当前的连接，可以用`quit`或者`exit`命令。  
Attach到目标进程上的arthas还会继续运行，端口会保持开放，下次连接时可以直接连接上。  
如果想完全退出，可以执行`shutdown`命令。

### 进阶使用 ###

本文只是起个抛砖引玉的效果，更多的功能可以查看官方文档  
[https://alibaba.github.io/arthas/index.html][https_alibaba.github.io_arthas_index.html]

### 实战案例 ###

这里还有些实战案例，阅读一下受益无穷  
[https://github.com/alibaba/arthas/issues?page=1&q=label%3Auser-case][https_github.com_alibaba_arthas_issues_page_1_q_label_3Auser-case]


[223523_49eb1786_955503]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20190726/64ab6a09-bc6a-4a2e-a54b-ca8a8bdbef45.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[http_127.0.0.1_8563]: http://127.0.0.1:8563/
[223523_946e6f5e_955503]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20190726/5a22dd7c-b683-4e2d-940b-01cd16f99375.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[dashboard]: https://alibaba.github.io/arthas/dashboard.html
[223523_e89912e5_955503]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20190726/9e55a9bf-1313-4490-a583-869e8e7c6b06.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[223524_3364fae6_955503]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20190726/f561c76e-3688-4888-ac8d-09b849088246.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[223524_a4f8523f_955503]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20190726/291cb63b-85e9-451c-9b40-e0c8480bf728.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[223524_97f8fdd0_955503]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20190726/62bf45f5-aad2-4fdc-a457-85abcd980965.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[https_alibaba.github.io_arthas_index.html]: https://alibaba.github.io/arthas/index.html
[https_github.com_alibaba_arthas_issues_page_1_q_label_3Auser-case]: https://github.com/alibaba/arthas/issues?page=1&q=label%3Auser-case