+++
title = "使用JMeter录制性能测试脚本"
date = "2018-08-28 06:43:00"
url = "archives/154"
tags = ["JMeter"]
categories = ["测试"]
+++

JMeter是一个开源的基于Java的性能测试工具，使用起来真的是即"方便"又"强大"

#### 新建线程组 ####

用于存放录制结果

#### 新建代理服务器 ####

1.  测试计划->新建非测试原件->HTTP代理服务器
2.  TestPlan Creation 将目标控制器设置为：测试计划>线程组 （录制后的请求信息将加到此线程组中来）
3.  Requests Filtering 请求过滤，排除无关的请求，具体配置如下：

包含模式：只录制指定主机的请求

```bash
.+(itkeeping.com).+
```

排除模式：排除静态请求

```bash
(?i).*.(bmp|css|js|gif|ico|jpe?g|png|swf|woff|woff2)
```

配置完成后截图如下：

![4d598591-f154-4f5e-93bc-45c1a8f79e6d.jpeg][]![3ec4f270-6191-4fc1-9404-dbdd37cbb2a0.jpeg][]

#### 录制前准备 ####

 *  启动代理服务器，默认端口8888。

![8e96a302-1bd3-44e9-9a27-09c595a637cb.jpeg][]

 *  启动代理服务器后，默认在bin目录下生成一个SSL证书，若不录制HTTPS网站，忽略此项。
 *  使用Firefox配置代理，以及SSL证书。

![7795e85a-45f3-4d8f-83e5-ab476aa1603c.jpeg][]![b7cafb33-6c3f-4ed6-982d-b8e228de5a08.jpeg][]

#### 录制 ####

一切准备就绪后，使用Firefox打开指定的网站进行操作，操作完成后，关闭代理服务器。

![4475b1b9-c4bc-492d-8938-e4c271542a57.jpeg][]

上图录制了从登陆到新建工单的整个过程, 有很多请求我现在并不需要，可以手动删除，只保留新建工单的接口。

#### 使用录制的脚本 ####

在使用之前，先进行一下线程组的相关设置，这里我设置了10个线程，共循环10次

![a5b2ed5b-e388-4c62-ad93-2b109fc56ec8.jpeg][]

另外我希望新建工单主题的uid和主题，描述每次都不一样，可以使用jmeter内置函数替换。

生成UUID：

```bash
${__UUID}
```

基于UUID生成30个随机字符：

```bash
${__RandomString(30,${__UUID},1)}
```

最终配置请求的参数如下：

![0a87e810-5c6b-4501-b57b-df9f1bcf1b01.jpeg][]

配置响应断言：更直观的判断请求是否成功，只要status=0就代表执行成功

![2a59b731-4cbe-4ee6-82d3-c3131609ba89.jpeg][]

启动一下试试吧！

结果查看树，断言全部通过，说明请求成功了

![10dc8725-ae60-435f-bd5b-eb382bbee7b7.jpeg][]

聚合报告，包含各项指标，反正就是没毛病：

![5e94ab47-cdf9-4e88-9839-e41d96b186a0.jpeg][]

最后看数据是否正常生成？

![f7fc40a5-654b-49c1-ab9d-c1f0c4d24200.jpeg][]


[4d598591-f154-4f5e-93bc-45c1a8f79e6d.jpeg]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20180828/4d598591-f154-4f5e-93bc-45c1a8f79e6d.jpeg?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[3ec4f270-6191-4fc1-9404-dbdd37cbb2a0.jpeg]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20180828/3ec4f270-6191-4fc1-9404-dbdd37cbb2a0.jpeg?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[8e96a302-1bd3-44e9-9a27-09c595a637cb.jpeg]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20180828/8e96a302-1bd3-44e9-9a27-09c595a637cb.jpeg?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[7795e85a-45f3-4d8f-83e5-ab476aa1603c.jpeg]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20180828/7795e85a-45f3-4d8f-83e5-ab476aa1603c.jpeg?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[b7cafb33-6c3f-4ed6-982d-b8e228de5a08.jpeg]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20180828/b7cafb33-6c3f-4ed6-982d-b8e228de5a08.jpeg?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[4475b1b9-c4bc-492d-8938-e4c271542a57.jpeg]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20180828/4475b1b9-c4bc-492d-8938-e4c271542a57.jpeg?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[a5b2ed5b-e388-4c62-ad93-2b109fc56ec8.jpeg]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20180828/a5b2ed5b-e388-4c62-ad93-2b109fc56ec8.jpeg?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[0a87e810-5c6b-4501-b57b-df9f1bcf1b01.jpeg]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20180828/0a87e810-5c6b-4501-b57b-df9f1bcf1b01.jpeg?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[2a59b731-4cbe-4ee6-82d3-c3131609ba89.jpeg]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20180828/2a59b731-4cbe-4ee6-82d3-c3131609ba89.jpeg?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[10dc8725-ae60-435f-bd5b-eb382bbee7b7.jpeg]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20180828/10dc8725-ae60-435f-bd5b-eb382bbee7b7.jpeg?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[5e94ab47-cdf9-4e88-9839-e41d96b186a0.jpeg]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20180828/5e94ab47-cdf9-4e88-9839-e41d96b186a0.jpeg?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[f7fc40a5-654b-49c1-ab9d-c1f0c4d24200.jpeg]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20180828/f7fc40a5-654b-49c1-ab9d-c1f0c4d24200.jpeg?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg