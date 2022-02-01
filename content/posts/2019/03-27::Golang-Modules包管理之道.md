+++
title = "Golang-Modules包管理之道"
date = "2019-03-27 10:11:00"
url = "archives/540"
tags = ["Golang"]
categories = ["后端"]
+++

golang原始的包管理方式非常low，然终于在`go version 1.1.1`之后，官方推出了模块概念，但是目前该功能还在试验阶段，有些地方还需要不断的进行完善。在官方正式宣布之前，打算不断修正这种支持。到时候就可以移除对`GOPATH`和`go get`命令的支持。

### 初始化 ###

目前modules机制仍在早期阶段，golang提供了一个环境变量“`GO111MODULE`”，默认值为auto

| on   | 如果你想直接使用modules而不需要从GOPATH过度，那么需要设置为on               |
|------|------------------------------------------------------|
| auto | 如果当前目录里有go.mod文件，就使用go modules，否则使用旧的GOPATH和vendor机制 |


modules和传统的GOPATH不同，不需要包含例如src，bin这样的子目录，一个源代码目录甚至是空目录都可以作为module，只要其中包含有go.mod文件。

我们就用一个空目录来创建我们的第一个module：

```bash
$ cd $GOPATH/src/github.com/wuwz
$ mkdir testmod
$ cd testmod
$ go mod init
go: creating new go.mod: module github.com/wuwz/testmod
```

初始完成后会在目录下生成一个go.mod文件：

```bash
module github.com/wuwz/testmod
```

### 包管理 ###

当我们使用go build，go test以及go list时，go会自动得更新go.mod文件，将依赖关系写入其中，为了验证这点，我们新建一个main.go并引入一个第三方包。

```go
$ vi main.go

package main

// 这里我们只是演示引入，但是不用使用，所以使用"_"忽略了编译检查
import (
  _ "github.com/wuwz/goddos"
  _ "golang.org/x/text"
)

fun main() {

}
```

然后执行: `go build`

```bash
$ go build
go: downloading github.com/wuwz/goddos latest
go: downloading golang.org/x/text v0.3.0

$ ls
go.mod  go.sum  main.go
```

我们注意到go自动下载了相关的两个依赖包，并且多个一个`go.sum`文件，同时`go.mod`文件被自动更新了：

```bash
$ cat go.mod
module github.com/wuwz/testmod

require (
    github.com/wuwz/goddos v0.0.0-20190321020003-f6b3fc254197
    golang.org/x/text v0.3.0
)

$ cat go.sum
github.com/gookit/color v1.1.6 h1:CisXBwYhzdPZUV+F8J4N3nzTclW78mOYz6TbPwUmhV4=
github.com/gookit/color v1.1.6/go.mod h1:655QfvFggjTrC1SaAufon2qad0RLgbdQa40lCeOdU64=
github.com/wuwz/goddos v0.0.0-20190321020003-f6b3fc254197 h1:ww/V0n1djPEqcmgBfzHPzxtoR2eJrMMri7IGa4haADI=
github.com/wuwz/goddos v0.0.0-20190321020003-f6b3fc254197/go.mod h1:eOQf8AQsIyLvHqlW//TEXORGpwt7r/0WLn93T0f+qcg=
golang.org/x/text v0.3.0 h1:g61tztE5qeGQ89tm6NTjjM9VPIm088od1l6aSorWRWg=
golang.org/x/text v0.3.0/go.mod h1:NqM8EUOU14njkJ3fqMW+pc6Ldnwhi/IjpwHt7yyuwOQ=
```

答案是显而易见的，我们直接引用package，go modules会根据这些去找到需要的包下载，然后写入文件，下次就会使用本次编译时使用的版本（基于github的tag分支，如果没有则是master分支）。

那么他的包下载到哪里去了呢？这次不在`$GOPATH/src`下了，而是`$GOPATH/pkg/mod`

![af301521-c363-4705-b8fd-30352af8add0.png][]

本地下载缓存后，下次就不需要再次到网上fetch了，虽然还远没有类似NPM的强大，但是聊胜于无嘛，哈哈！

值得一提的是，在当前项目下使用go get命令也会将依赖写入go.mod文件，反之你也可以自行修改go.mod文件。

### go mod 命令一览 ###

| go mod download | 下载模块到本地缓存，具体可以通过命令go env查看，其中环境变量GOCACHE就是缓存的地址，如果该文件夹的内容太大，可以通过命令go clean -cache |
|-----------------|-----------------------------------------------------------------------------------|
| go mod edit     | 从工具或脚本中编辑go.mod文件                                                                 |
| go mod graph    | 打印模块需求图                                                                           |
| go mod init     | 在当前目录下初始化新的模块                                                                     |
| go mod tidy     | 添加缺失的模块以及移除无用的模块                                                                  |
| go mod verify   | 验证依赖项是否达到预期的目的                                                                    |
| go mod why      | 解释为什么需要包或模块                                                                       |



[af301521-c363-4705-b8fd-30352af8add0.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20190327/af301521-c363-4705-b8fd-30352af8add0.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg