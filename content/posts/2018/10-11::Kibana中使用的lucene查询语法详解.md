+++
title = "Kibana中使用的lucene查询语法详解"
date = "2018-10-11 06:05:00"
url = "archives/100"
tags = ["Kibana","Lucene"]
categories = ["运维"]
+++

Kibana是一个分析和可视化平台，可用来搜索、查看、交互存放在Elasticsearch索引里的数据

![c53c68ba-8c4c-4db2-a7fa-2df160c3ba4b.png][]

本文简单概括在搜索框中使用lucene查询语法检索相关的日志数据。

### 全文搜索 ###

直接输入关键字，将返回所有字段值中包含关键字的文档：

![e52154cb-04d5-4fab-863b-e68bca2247f9.png][]

使用双引号包起来作为一个短语搜索精准匹配：

```bash
"providerId=719"
```

![e7b621d4-4ab8-433c-997d-737d536864c1.png][]

### 字段 ###

可以直接通过页面配置：

![df295e45-9e1e-4871-8904-753feb5895d7.png][]

同时也可以在输入框中输入相关语法：

```bash
className:com.ewei.module.talk.logic.impl.ChatLogicImpl
```

![18ea9315-807b-4b63-b3d8-41118c1ee6db.png][]

相关的语法说明：

 *  限定字段全文搜索：field:value
 *  限定字段精确搜索：field:"value"

如：`level:error` 表示搜索基本为error的日志信息

 *  字段本身是否存在：*exists*:field
 *  不能包含某个字段：*missing*:field

另外，大概有哪些字段可以使用呢？见下图：

![0737495f-c37b-4157-a71a-9516789ed652.png][]

### 通配 ###

```bash
? 匹配单个字符
* 匹配0到多个字符

如：kiba?a, el*search

注意，? * 不能用作第一个字符，例如：?text *text
```

### 正则 ###

es支持部分正则功能,性能较差

```bash
name:/joh?n(ath[oa]n)/
```

### 模糊搜索 ###

```bash
quikc~ brwn~ foks~

~:在一个单词后面加上~启用模糊搜索，可以搜到一些拼写错误的单词

first~ 这种也能匹配到 frist

还可以设置编辑距离（整数），指定需要多少相似度
cromm~1 会匹配到 from 和 chrome
默认2，越大越接近搜索的原始值，设置为1基本能搜到80%拼写错误的单词
```

### 近似搜索 ###

```bash
在短语后面加上~，可以搜到被隔开或顺序不同的单词
"where select"~5 表示 select 和 where 中间可以隔着5个单词，可以搜到 select password from users where id=1
```

### 范围搜索 ###

```bash
数值/时间/IP/字符串 类型的字段可以对某一范围进行查询
length:[100 TO 200]
sip:["172.24.20.110" TO "172.24.20.140"]
date:{"now-6h" TO "now"}
tag:{b TO e} 搜索b到e中间的字符
count:[10 TO *] * 表示一端不限制范围
count:[1 TO 5} [ ] 表示端点数值包含在范围内，{ } 表示端点数值不包含在范围内，可以混合使用，此语句为1到5，包括1，不包括5
可以简化成以下写法：
age:>10
age:=10 AND  < ! ( ) { } [ ] ^ " ~ * ? :  /
以上字符当作值搜索的时候需要用转义
(1+1)=2用来查询(1+1)=2
```


[c53c68ba-8c4c-4db2-a7fa-2df160c3ba4b.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20181011/c53c68ba-8c4c-4db2-a7fa-2df160c3ba4b.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[e52154cb-04d5-4fab-863b-e68bca2247f9.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20181011/e52154cb-04d5-4fab-863b-e68bca2247f9.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[e7b621d4-4ab8-433c-997d-737d536864c1.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20181011/e7b621d4-4ab8-433c-997d-737d536864c1.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[df295e45-9e1e-4871-8904-753feb5895d7.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20181011/df295e45-9e1e-4871-8904-753feb5895d7.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[18ea9315-807b-4b63-b3d8-41118c1ee6db.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20181011/18ea9315-807b-4b63-b3d8-41118c1ee6db.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[0737495f-c37b-4157-a71a-9516789ed652.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20181011/0737495f-c37b-4157-a71a-9516789ed652.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg