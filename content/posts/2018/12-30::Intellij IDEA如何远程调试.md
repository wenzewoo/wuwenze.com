+++
title = "Intellij IDEA如何远程调试"
date = "2018-12-30 06:21:00"
url = "archives/126"
tags = ["Intellij IDEA"]
categories = ["开发工具"]
+++

一般情况下，对于分布式系统的调试还是比较麻烦的，比较常见的方式是在远程调用的过程中通过不断的打印log，然后重新部署上线、调试、定位问题，实在是过于麻烦。

实际上Java是支持远程调试的，只是大家平时没有怎么用过罢了，本文通过`Intellij IDEA`为例讲解如何来使用远程调试。

### 准备测试程序 ###

```java
@GetMapping("/list")
public ResponseEntity<?> list() {
    final List<String> arrayList = new ArrayList<>();
    for (int i = 0; i < 1000; i++)
        arrayList.add(String.format("arrayList_item_%s", i));
    return ResponseEntity.ok(arrayList);
}
```

这个程序很简单，就是循环生成ArrayList对象罢了。

### 使用特定的JVM参数启动程序 ###

将程序上传到服务器（`10.211.55.5`）上后，使用相关的JVM参数启动程序:

```bash
-Xdebug -Xrunjdwp:transport=dt_socket,address=${debugger_port},server=y,suspend=n
```

其中，`${debugger_port}`代表开启远程调试的端口，开启完毕后，要注意防火墙相关的配置

```bash
java -D -Xdebug -Xrunjdwp:transport=dt_socket,address=9012,server=y,suspend=n -jar springboot-test-0.0.1-SNAPSHOT.jar
```

启动成功后如图：

![d75a49ec-0fbc-44de-adef-6418a58f5974.png][]

### 配置IDEA连接远程调试 ###

打开`Edit Configurations`，然后新建`Remote` 配置后如图：

![c9b30ff0-6953-474f-bb51-31bea1b8fab0.png][]

然后启动之，如若控制台出现以下提示，则表示连接成功了：

```bash
Connected to the target VM, address: '10.211.55.5:9012', transport: 'socket'
```

### 开始调试 ###

后面的步骤就像是在本地一样了，该怎么调试就怎么调试：

![deecdfd2-cfa2-409d-8c71-dd5e98eb5989.png][]


[d75a49ec-0fbc-44de-adef-6418a58f5974.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20181230/d75a49ec-0fbc-44de-adef-6418a58f5974.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[c9b30ff0-6953-474f-bb51-31bea1b8fab0.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20181230/c9b30ff0-6953-474f-bb51-31bea1b8fab0.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[deecdfd2-cfa2-409d-8c71-dd5e98eb5989.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20181230/deecdfd2-cfa2-409d-8c71-dd5e98eb5989.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg