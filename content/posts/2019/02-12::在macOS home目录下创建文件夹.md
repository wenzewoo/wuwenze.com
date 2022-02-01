+++
title = "在macOS /home目录下创建文件夹"
date = "2019-02-12 07:59:00"
url = "archives/467"
tags = ["macOS"]
categories = ["杂项"]
+++

macOS 基于`unix`， 自带就有`/home`目录，但是为空。`/home`目录的默认所属用户是root wheel，默认的root账号所属用户是root admin，所以root也无法在home目录下创建文件夹。如果非要使用home目录，下面会详细说明(**备注：个人不建议使用home目录**)

#### 修改auto\_master ####

```bash
$ sudo vim /etc/auto_master 
### 
### Automounter master map 
### 
+auto_master        ### Use directory service 
/net            -hosts      -nobrowse,hidefromfinder,nosuid 
##/home auto_home -nobrowse,hidefromfinder //注释掉本行 
/Network/Servers    -fstab 
/-          -static
```

#### **加载auto\_master** ####

```bash
$ cd /    //必须切换到根目录 
$ sudo automount  //必须在根目录下执行
```

#### **创建目录与修改权限** ####

```bash
$ sudo mkdir /home/test //创建目录 
$ cd /home 
$ sudo chown wuwenze.staff -R test  //修改文件所属 
$ ls -l /home/ 
total 0 
dr-xr-xr-x 7 root wheel 238 2 26 17:48 ./ 
drwxr-xr-x 30 root wheel 1088 8 30 17:28 ../ 
drwxr-xr-x 2 wuwenze staff 68 2 26 17:45 test/
```