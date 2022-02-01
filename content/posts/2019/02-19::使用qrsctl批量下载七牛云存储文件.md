+++
title = "使用qrsctl批量下载七牛云存储文件"
date = "2019-02-19 08:11:00"
url = "archives/479"
tags = ["七牛","Shell"]
categories = ["杂项"]
+++

由于Markdown文档图床需要，之前选用了七牛作为云存储，但是前几天突然发现我所有的图片外链全部失效了，  
原来是七牛将测试域名回收了，同时我自己的已备案域名也已经过期，导致我存储在七牛中的所有图片既不能预览，也不能下载，甚是恶心，在七牛的官网翻了一圈，总算是找到了把所有文件下载下来的解决方案。

![25d58662-e77c-4ee5-932a-d0982ef6c888.png][]

### 安装命令行辅助工具(qrsctl) ###

下载地址：[https://developer.qiniu.com/kodo/tools/1300/qrsctl][https_developer.qiniu.com_kodo_tools_1300_qrsctl]

![38dbef41-937b-41f1-8302-b66565e59bfb.png][]

我这里使用的macOS，其他系统大同小异，参考着来吧。

```bash
cd ~/Downloads
wget http://devtools.qiniu.com/darwin/amd64/qrsctl

#### 赋予qrsctl可执行权限
chmod +x qrsctl

#### 执行测试
./qrsctl
```

![2d763548-fa12-420e-a337-126ec4023898.png][]

出现如图所示的文档提示，表示已经配置好了，至于要不要加到`/usr/local/bin`中，就没有必要了，临时用一下嘛

### 登陆 ###

```bash
./qrsctl login <User> <Passwd>
```

### 查询buckets ###

```bash
./qrsctl buckets
```

![51c1db4a-5add-4ea5-afb0-cef38ad52c8a.png][]

得到当前账号下的所有存储空间后，记住名字，后面就会用到拿来下载了。

### 查询文件清单 ###
```bash
./qrsctl listprefix <Bucket Name> ''
```

![ce60c3a3-f979-4c30-8154-bcbabb90fc58.png][]

### 下载指定文件 ###

```bash
##./qrsctl get <Bucket Name> <File Name> <Dest File>
### 下载指定文件
./qrsctl get filestore 2018-08-28-16-54-52.jpg ~/Downloads/2018-08-28-16-54-52.jpg
```

### 批量下载脚本 ###

按照以上的流程，一次只能下载一个文件，简单的写个脚本来完成批量下载吧

```bash
##!/bin/bash 
imgs=`./qrsctl listprefix filestore ''`

i=0 
echo $imgs | tr " " "n" | while read line
do
    if(($i>0))
    then
        echo $line
        ./qrsctl get filestore $line ./$line
    fi
    i=$(($i+1))
done
```

给脚本赋予可执行权限，执行后就开始自动下载了

> 这个脚本达不到全自动的目的，但是将所有文件下载下来还是妥妥的，等下载完成后，就告别这个坑爹的七牛吧～


[25d58662-e77c-4ee5-932a-d0982ef6c888.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20190219/25d58662-e77c-4ee5-932a-d0982ef6c888.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[https_developer.qiniu.com_kodo_tools_1300_qrsctl]: https://developer.qiniu.com/kodo/tools/1300/qrsctl
[38dbef41-937b-41f1-8302-b66565e59bfb.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20190219/38dbef41-937b-41f1-8302-b66565e59bfb.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[2d763548-fa12-420e-a337-126ec4023898.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20190219/2d763548-fa12-420e-a337-126ec4023898.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[51c1db4a-5add-4ea5-afb0-cef38ad52c8a.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20190219/51c1db4a-5add-4ea5-afb0-cef38ad52c8a.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[ce60c3a3-f979-4c30-8154-bcbabb90fc58.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20190219/ce60c3a3-f979-4c30-8154-bcbabb90fc58.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg