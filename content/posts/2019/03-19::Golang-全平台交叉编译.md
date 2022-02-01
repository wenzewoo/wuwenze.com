+++
title = "Golang-全平台交叉编译"
date = "2019-03-19 10:09:00"
url = "archives/538"
tags = ["Golang"]
categories = ["后端"]
+++

Golang的交叉编译要保证golang版本在1.5以上，我的环境是`go1.11.5 darwin/amd64`

#### 交叉编译 ####

```bash
### mac上编译linux和windows二进制
CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build main.go
CGO_ENABLED=0 GOOS=windows GOARCH=amd64 go build main.go

### linux上编译mac和windows二进制
CGO_ENABLED=0 GOOS=darwin GOARCH=amd64 go build main.go
CGO_ENABLED=0 GOOS=windows GOARCH=amd64 go build main.go

### windows上编译mac和linux二进制
SET CGO_ENABLED=0 SET GOOS=darwin SET GOARCH=amd64 go build main.go
SET CGO_ENABLED=0 SET GOOS=linux SET GOARCH=amd64 go build main.go
```

#### 参数解析 ####

 *  GOOS：目标操作系统
 *  GOARCH：目标操作系统的架构

| OS      | ARCH              | OS version                 |
|---------|-------------------|----------------------------|
| linux   | 386 / amd64 / arm | >= Linux 2.6               |
| darwin  | 386 / amd64       | OS X (Snow Leopard + Lion) |
| freebsd | 386 / amd64       | >= FreeBSD 7               |
| windows | 386 / amd64       | >= Windows 2000            |


编译其他平台的时候根据上面表格参数执行编译就可以了。

#### CGO ####

CGO\_ENABLED=0的意思是使用C语言版本的GO编译器，参数配置为0的时候就关闭C语言版本的编译器了。自从golang1.5以后go就使用go语言编译器进行编译了。在golang1.9当中没有使用CGO\_ENABLED参数发现依然可以正常编译。当然使用了也可以正常编译。比如把CGO\_ENABLED参数设置成1，即在编译的过程当中使用CGO编译器，我发现依然是可以正常编译的。  
实际上如果在go当中使用了C的库，比如`import "C"`默认使用go build的时候就会启动CGO编译器，当然我们可以使用CGO\_ENABLED=0来控制go build是否使用CGO编译器。