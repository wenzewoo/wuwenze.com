+++
title = "Ubuntu下使用Shadowsocks GFWList"
date = "2018-07-26 07:28:00"
url = "archives/427"
tags = ["Linux","Shadowsocks","GFWList"]
categories = ["杂项"]
+++

现有的SS客户端在Linux上仅支持全局代理，本文以Ubuntu发行版为例，配置PAC自动代理，达到无缝切换的目的。

#### 更新系统 ####

```bash
$ sudo apt update
$ sudo apt upgrade
```

#### 安装Shadowsocks GUI ####

> 非ubuntu系统可以参考此链接自行编译

[https://github.com/shadowsocks/shadowsocks-qt5][https_github.com_shadowsocks_shadowsocks-qt5]

在ubuntu上安装相当简单，可直接使用PPA源（`14.04 lts以上系统`）

##### 安装相关依赖 #####

```bash
$ sudo apt install libappindicator1 libindicator7
```

##### 安装shadowsocks-qt5 #####

```bash
$ sudo add-apt-repository ppa:hzwhuang/ss-qt5
$ sudo apt-get update
$ sudo apt-get install shadowsocks-qt5
```

#### 配置Shadowsocks服务 ####

配置过程不做过多描述, 自行购买相关服务后配置, 配置完成后如下图所示：

![c63b6fbe-c108-41a4-a62a-abfc702f374a.png][]

#### 配置网络代理（全局） ####

![764c4a19-a2e7-4392-8154-0d4dd9f008b6.png][]

此时，所有的HTTP请求都将通过代理，显然不是想要的结果；

#### 配置基于gfwlist的pac文件生成工具 ####

> 什么是gfwlist? [https://github.com/gfwlist/gfwlist][https_github.com_gfwlist_gfwlist]

什么是PAC? [https://baike.baidu.com/item/PAC/16292100][https_baike.baidu.com_item_PAC_16292100]

##### 1. 安装pip #####

```bash
$ sudo apt install python-pip
$ pip install --upgrade pip
```

##### 2. 安装GenPAC #####

```bash
$ sudo pip install genpac
$ pip install --upgrade genpac
```

##### 3. 使用GenPAC生成pac文件(基于gfwlist) #####

GenPAC: [https://github.com/JinnLynn/GenPAC][https_github.com_JinnLynn_GenPAC]

```bash
$ genpac -p "SOCKS5 127.0.0.1:1080" --gfwlist-proxy="SOCKS5 127.0.0.1:1080" --gfwlist-url=https://raw.githubusercontent.com/gfwlist/gfwlist/master/gfwlist.txt --output="autoproxy.pac"
```

生成文件位于当前执行命令路径(我的生成为：/home/ubuntu/autoproxy.pac),文件内容如下：

![9e2e952f-8544-4962-b4cb-a8b95628e4e5.png][]

#### 配置网络代理（自动PAC） ####

![fea03dd6-5fa6-47b8-a2e4-31c0724247f9.png][]

参考上图配置网络代理

| 方法    | 自动                   |
|-------|----------------------|
| 配置URL | file://{pacFilePath} |



[https_github.com_shadowsocks_shadowsocks-qt5]: https://github.com/shadowsocks/shadowsocks-qt5
[c63b6fbe-c108-41a4-a62a-abfc702f374a.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20180726/c63b6fbe-c108-41a4-a62a-abfc702f374a.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[764c4a19-a2e7-4392-8154-0d4dd9f008b6.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20180726/764c4a19-a2e7-4392-8154-0d4dd9f008b6.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[https_github.com_gfwlist_gfwlist]: https://github.com/gfwlist/gfwlist
[https_baike.baidu.com_item_PAC_16292100]: https://baike.baidu.com/item/PAC/16292100
[https_github.com_JinnLynn_GenPAC]: https://github.com/JinnLynn/GenPAC
[9e2e952f-8544-4962-b4cb-a8b95628e4e5.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20180726/9e2e952f-8544-4962-b4cb-a8b95628e4e5.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[fea03dd6-5fa6-47b8-a2e4-31c0724247f9.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20180726/fea03dd6-5fa6-47b8-a2e4-31c0724247f9.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg