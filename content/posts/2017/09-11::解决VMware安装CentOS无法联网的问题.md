+++
title = "解决VMware安装CentOS无法联网的问题"
date = "2017-09-11 07:38:00"
url = "archives/437"
tags = ["VMWare"]
categories = ["杂项"]
+++

由于centos 6.5 minimal默认没有开启网卡，所以需要手动配置一下；

### 导致的问题 ###

1.  无法与主机通讯；
2.  无法连接外网；

![099e5efe-b694-4baa-b6f9-b6b233c58c42.png][]

### 配置步骤 ###

#### 1) 确认虚拟机是否使用`NAT模式`： ####

![955212dd-8fe4-4807-9abf-88b099f0c08a.png][]

#### 2) 记录一下网关、网段信息：进入菜单（`编辑`\-`虚拟网络编辑器`） ####

![b232e954-53d6-44b3-a80c-4c585d53b895.png][]

#### 3) 选择`VMnet8 (NAT模式)`，`取消DHCP服务勾选`，然后点击NAT设置： ####

![b94999fb-60d9-41f1-913e-3d3e879533a4.png][]

> 记录一下网关地址（`192.168.253.2`）后面会用到的

#### 4) 打开`网络和共享中心` - `更改适配器设置` - 选择`VMware Network Adapter VMnet8` - 右键状态 - 详细信息 ####

![c744f550-0d87-4b3e-87d7-6256175474cc.png][]

> 其中，IPv4地址是虚拟路由器为Windows分配的地址，IPv4 WINS 服务器是虚拟路由器的网关地址

#### 5) 进入CentOS，进行网络配置： ####

```bash
/etc/sysconfig/network-scripts/ifcfg-eth0
```

![a0080fcc-91f9-4b1a-88c6-76dd264fa5bf.png][]

```bash
#### 修改的内容
ONBOOT=yes ##自启动,默认为no
NM_CONTROLLED=no ##不需要,关闭
BOOTPROTO=static ##默认为dbcp, 修改为静态
 
#### 添加的内容
IPADDR=192.168.253.110 ##根据网关设置的静态IP
NETMASK=255.255.255.0
GATEWAY=192.168.253.2 ##网关地址
```

#### 6) 重启网卡 ####

```bash
service network restart
```

![968a6c19-b893-420c-a06e-85fcddfd2499.png][]

#### 7) 检测是否能够与主机进行通讯 ####

![4eead5f1-2eed-4e8c-971a-595dc0a61878.png][]

#### 8) 设置DNS服务器 ####

```bash
vi /etc/resolv.conf
nameserver 114.114.114.114
```

#### 9) 检测是否能够连接外网 ####

![5f5766a7-bce3-47a8-b1d3-5daaed39b4d0.png][]


[099e5efe-b694-4baa-b6f9-b6b233c58c42.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20170911/099e5efe-b694-4baa-b6f9-b6b233c58c42.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[955212dd-8fe4-4807-9abf-88b099f0c08a.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20170911/955212dd-8fe4-4807-9abf-88b099f0c08a.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[b232e954-53d6-44b3-a80c-4c585d53b895.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20170911/b232e954-53d6-44b3-a80c-4c585d53b895.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[b94999fb-60d9-41f1-913e-3d3e879533a4.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20170911/b94999fb-60d9-41f1-913e-3d3e879533a4.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[c744f550-0d87-4b3e-87d7-6256175474cc.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20170911/c744f550-0d87-4b3e-87d7-6256175474cc.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[a0080fcc-91f9-4b1a-88c6-76dd264fa5bf.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20170911/a0080fcc-91f9-4b1a-88c6-76dd264fa5bf.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[968a6c19-b893-420c-a06e-85fcddfd2499.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20170911/968a6c19-b893-420c-a06e-85fcddfd2499.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[4eead5f1-2eed-4e8c-971a-595dc0a61878.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20170911/4eead5f1-2eed-4e8c-971a-595dc0a61878.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[5f5766a7-bce3-47a8-b1d3-5daaed39b4d0.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20170911/5f5766a7-bce3-47a8-b1d3-5daaed39b4d0.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg