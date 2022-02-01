+++
title = "解决macOS Homebrew一直卡在Updating的问题"
date = "2019-02-06 07:53:00"
url = "archives/463"
tags = ["macOS"]
categories = ["杂项"]
+++

在国内的网络环境下使用 `Homebrew` 安装软件的过程中可能会长时间卡在 `Updating Homebrew` 这个步骤。

![7c2e803b-34b2-488f-8cc6-a34b3eb2b41b.png][]

每次执行`brew install`命令时，会尝试更新`Homebrew`，但是由于众所周知的原因，这一步被挡在了墙外，本文有两种方式可解决此问题。

### 临时解决 ###

出现此提示时，轻按`Control + C`命令终止更新操作。

```bash
~ brew install macvim
Updating Homebrew...
^C==> Satisfying dependencies
==> ....
```

这个方法是临时性的，每次都去按一下也是神烦。

### 使用Alibaba加速镜像 ###

#### 1) 替换/还原 brew.git 仓库地址 ####

```bash
### 替换成阿里巴巴的 brew.git 仓库地址:
cd "$(brew --repo)"
git remote set-url origin https://mirrors.aliyun.com/homebrew/brew.git

##=======================================================

### 还原为官方提供的 brew.git 仓库地址
cd "$(brew --repo)"
git remote set-url origin https://github.com/Homebrew/brew.git
```

#### 2) 替换/还原 homebrew-core.git 仓库地址 ####

```bash
### 替换成阿里巴巴的 homebrew-core.git 仓库地址:
cd "$(brew --repo)/Library/Taps/homebrew/homebrew-core"
git remote set-url origin https://mirrors.aliyun.com/homebrew/homebrew-core.git

##=======================================================

### 还原为官方提供的 homebrew-core.git 仓库地址
cd "$(brew --repo)/Library/Taps/homebrew/homebrew-core"
git remote set-url origin https://github.com/Homebrew/homebrew-core.git
```

#### 3) 替换/还原 homebrew-bottles 访问地址 ####

这个步骤跟你的 macOS 系统使用的 shell 版本有关系，所以，先来查看当前使用的 shell 版本：

```bash
echo $SHELL
### /bin/zsh or /bin/bash
```

##### 3.1) zsh 终端操作方式 #####

```bash
### 替换成阿里巴巴的 homebrew-bottles 访问地址:
echo 'export HOMEBREW_BOTTLE_DOMAIN=https://mirrors.aliyun.com/homebrew/homebrew-bottles' >> ~/.zshrc
source ~/.zshrc

##=======================================================

### 还原为官方提供的 homebrew-bottles 访问地址
vi ~/.zshrc
### 然后，删除 HOMEBREW_BOTTLE_DOMAIN 这一行配置
source ~/.zshrc
```

##### 3.2) bash 终端操作方式 #####

```bash
### 替换 homebrew-bottles 访问 URL:
echo 'export HOMEBREW_BOTTLE_DOMAIN=https://mirrors.aliyun.com/homebrew/homebrew-bottles' >> ~/.bash_profile
source ~/.bash_profile

##=======================================================

### 还原为官方提供的 homebrew-bottles 访问地址
vi ~/.bash_profile
### 然后，删除 HOMEBREW_BOTTLE_DOMAIN 这一行配置
source ~/.bash_profile
```


[7c2e803b-34b2-488f-8cc6-a34b3eb2b41b.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20190206/7c2e803b-34b2-488f-8cc6-a34b3eb2b41b.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg