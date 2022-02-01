+++
title = "Golang开发环境配置"
date = "2019-02-19 08:03:00"
url = "archives/470"
tags = ["Golang"]
categories = ["后端"]
+++

现如今Go语言的开发阵容可以说是空前强大，且背靠Google这棵大树，又不乏牛人坐镇，是名副其实的“牛二代”。  


![93bb4268-aba8-4e9e-adb8-2eb516ab4d16.png][]

有关Go语言特性优缺点本文就不再赘述了，百度上一大堆。 

### 下载SDK ###

> 本文使用macOS操作系统为例，其他系统操作大同小异

下载地址：[https://golang.org/dl/][https_golang.org_dl]

![34b9f2fa-5667-4940-b001-5e532d5bf020.png][]

### 安装SDK ###

双击下载后的`pkg`进行安装，安装成功后，打开终端，输入下面命令查看是否安装成功

![bcb87be5-0483-40f3-9346-6bb378351469.png][]

### 环境变量配置 ###

```bash
export GOPATH=/Users/wuwenze/Development/GoProjects
export GOBIN=$GOPATH/bin
export PATH=$PATH:$GOBIN
```

将以上内容写入到环境变量配置文件中（`~/.bash_profile`）  


![8ddcbdf2-c225-4127-b1b3-b9b4563c1979.png][]

需要注意的是，如果你的macOS使用的是zsh终端，系统会加载`~/.zshrc`文件，并不会加载.bash\_profile，所以需要在~/.zshrc中追加以下语句，使.bash\_profile文件生效：

```bash
source ~/.bash_profile
```

![da4b87ef-262e-4255-bb5b-49d65e87be06.png][]

#### 环境变量说明 ####

 *  GOPATH：开发常用文件夹，可以配置多个。该文件夹下有3个文件夹（src/pkg/bin）
    
     *  src：存放源代码文件
     *  pkg：编译后生成的文件(`.a`文件)（非main函数的文件在`go install`后生成）
     *  bin：存放编译后生成的可执行文件，可以自己执行
 *  GOBIN：是GOPATH下的bin目录

### Hello World ###

```bash
cd ~/Development/GoProjects/src
nano hello.go
```

编写Hello World源代码：

```go
package main

import "fmt"

func main() {
  fmt.Println("Hello World!")
}
```

编译运行：

```bash
go run hello.go
```

![584d559b-a42d-4c38-9204-381d08116ba7.png][]


[93bb4268-aba8-4e9e-adb8-2eb516ab4d16.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20190219/93bb4268-aba8-4e9e-adb8-2eb516ab4d16.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[https_golang.org_dl]: https://golang.org/dl/
[34b9f2fa-5667-4940-b001-5e532d5bf020.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20190219/34b9f2fa-5667-4940-b001-5e532d5bf020.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[bcb87be5-0483-40f3-9346-6bb378351469.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20190219/bcb87be5-0483-40f3-9346-6bb378351469.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[8ddcbdf2-c225-4127-b1b3-b9b4563c1979.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20190219/8ddcbdf2-c225-4127-b1b3-b9b4563c1979.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[da4b87ef-262e-4255-bb5b-49d65e87be06.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20190219/da4b87ef-262e-4255-bb5b-49d65e87be06.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[584d559b-a42d-4c38-9204-381d08116ba7.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20190219/584d559b-a42d-4c38-9204-381d08116ba7.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg