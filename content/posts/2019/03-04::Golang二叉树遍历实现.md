+++
title = "Golang二叉树遍历实现"
date = "2019-03-04 09:58:00"
url = "archives/508"
tags = ["Golang"]
categories = ["后端"]
+++

### 二叉树的几种遍历方式 ###

#### 1）先序遍历 ####

1)访问根节点；2)采用先序递归遍历左子树；3)采用先序递归遍历右子树；

![f2e0081d-247a-4651-acf3-4c6828162db4.png][]

  
(注：每个节点的分支都遵循上述的访问顺序，体现“递归调用”)

#### 2）中序遍历 ####

1)采用中序遍历左子树；2)访问根节点；3)采用中序遍历右子树；

![c0a7f813-9345-4f89-b790-ff384b54e32f.png][]

#### 3）后序遍历 ####

1)采用后序递归遍历左子树；2)采用后序递归遍历右子树；3)访问根节点；

![2019-08-19-073608][]

三种方法遍历过程中经过节点的路线一样，只是访问各个节点的时机不同。  
递归算法主要使用堆栈来实现，本文使用Golang实现，就当是Golang学习的练手题了。

### 使用Golang实现 ###

#### 定义结构体 ####

```go
type Node struct {
    Value int
    Left, Right *Node
}
```

#### 建树 ####

新建一个`node_test.go`文件，用于单元测试：

```go
// 按照图示，创建相关的二叉树
func CreateNode() *Node {
    root := &Node{'A', nil, nil}
    root.Left = &Node{Value: 'B'}
    root.Right = &Node{Value: 'C'}
    root.Left.Left = &Node{Value: 'D'}
    root.Left.Right = &Node{Value: 'F'}
    root.Left.Right.Left = new(Node)
    root.Left.Right.Left.Value = 'E'

    root.Right.Left = &Node{Value: 'G'}
    root.Right.Left.Right = &Node{Value: 'H'}
    root.Right.Right = &Node{Value: 'I'}
    return root
}
```

#### 几种遍历方法的实现 ####

继续在结构体中定义方法，先实现效果

```go
// 先序遍历
func (node *Node) TraversingWithPreface()  {
    if node != nil {
        node.PrintValue() // 先打印值，后续优化
        node.Left.TraversingWithPreface()
        node.Right.TraversingWithPreface()
    }
}

// 中序遍历
func (node *Node) TraversingWithMediumOrder()  {
    if node != nil {
        node.Left.TraversingWithMediumOrder()
        node.PrintValue() // 先打印值，后续优化
        node.Right.TraversingWithMediumOrder()
    }
}

// 后序遍历
func (node *Node) TraversingWithPostOrder()  {
    if node != nil {
        node.Left.TraversingWithPostOrder()
        node.Right.TraversingWithPostOrder()
        node.PrintValue() // 先打印值，后续优化
    }
}
```

编写单元测试

```go
func TestTraversing(t *testing.T) {
    node := CreateNode()

    fmt.Println("TraversingWithPreface::")
    node.TraversingWithPreface()
    fmt.Println()

    fmt.Println("TraversingWithMediumOrder::")
    node.TraversingWithMediumOrder()
    fmt.Println()


    fmt.Println("TraversingWithPostOrder::")
    node.TraversingWithPostOrder()
    fmt.Println()
}
```
    
```bash
=== RUN   TestTraversing
TraversingWithPreface::
(A) (B) (D) (F) (E) (C) (G) (H) (I) 
TraversingWithMediumOrder::
(D) (B) (E) (F) (A) (G) (H) (C) (I) 
TraversingWithPostOrder::
(D) (E) (F) (B) (H) (G) (I) (C) (A) 
--- PASS: TestTraversing (0.00s)
PASS
```

通过实现看到，遍历二叉树相当简单，只是打印的时机不同而已，上述的实现方法非常不妥，只能打印，遍历出来没法使用咋个办？继续优化吧

#### 使用channel + func继续优化 ####

上述的几种遍历方式，留下比较常用的中序遍历即可    
然后我们希望在递归调用的过程中，将遍历到的值通过channel发送给外面使用，直接看代码吧！

```go
// 在Golang的世界中，func和channel均为一等公民，均可以作为返回值、参数等
// 利用该特性，在此我们分别通过两种方式实现了将遍历到的值传递到调用方处理
package tree

import "fmt"

type Node struct {
    Value int
    Left, Right *Node
}

func (node *Node) PrintValue() {
    fmt.Printf("(%c)", node.Value)
}

func (node *Node) TraversingWithFunc(applyFunc func(*Node))  {
    if node != nil {
        node.Left.TraversingWithFunc(applyFunc)
        applyFunc(node)
        node.Right.TraversingWithFunc(applyFunc)
    }
}

func (node *Node) TraversingWithChannel() chan *Node {
    channel := make(chan *Node)
    go func() {
        node.TraversingWithFunc(func(node *Node) {
            channel <- node
        })
        close(channel)
    }()
    return channel
}
```

通过单元测试，来打印二叉树的以及统计总节点数：

```go
func TestTraversing(t *testing.T) {
    node := CreateNode()

    sumNodeCount := 0
    beginTime := time.Now()
    node.TraversingWithFunc(func(node *Node) {
        node.PrintValue()
        sumNodeCount ++
    })
    fmt.Printf("nnode.TraversingWithFunc sumNodeCount = %d, timeConsuming=%dnn", sumNodeCount, time.Now().Sub(beginTime))

    sumNodeCount = 0
    beginTime = time.Now()
    channel := node.TraversingWithChannel()
    for node := range channel {
        node.PrintValue()
        sumNodeCount ++
    }
    fmt.Printf("nnode.TraversingWithChannel sumNodeCount = %d, timeConsuming=%dn", sumNodeCount, time.Now().Sub(beginTime))
}
```

打印日志如下：

```bash
=== RUN   TestTraversing
(D) (B) (E) (F) (A) (G) (H) (C) (I) 
node.TraversingWithFunc sumNodeCount = 9, timeConsuming=50194

(D) (B) (E) (F) (A) (G) (H) (C) (I) 
node.TraversingWithChannel sumNodeCount = 9, timeConsuming=49098
--- PASS: TestTraversing (0.00s)
PASS
```

可以看到，由于TraversingWithChannel()使用了goroutine+channel实现并发模型，目前看来效率远高于前者。

将相关的耗时统计代码去掉，进行进一步的性能测试：

```go
func BenchmarkNode_TraversingWithFunc(b *testing.B) {
    node := CreateNode()
    for i := 0 ; i < b.N; i ++ {
        node.TraversingWithFunc(func(node *Node) {
            node.PrintValue()
        })
        fmt.Println()
    }
}

func BenchmarkNode_TraversingWithChannel(b *testing.B) {
    node := CreateNode()
    for i := 0 ; i < b.N; i ++ {
        channel := node.TraversingWithChannel()
        for node := range channel {
            node.PrintValue()
        }
        fmt.Println()
    }
}
```

![2019-08-19-073609][]![2019-08-19-073611][]

从图中可以看到，分别对`TraversingWithFunc()`/ `TraversingWithChannel()`做了 3w / 5w次的性能测试，仍然是后者领先。

#### 使用pprof进行性能调优 ####

以上的代码是否能分析出一些性能瓶颈，具体耗时是什么操作产生的呢？答案是肯定的，使出Golang的大杀器pprof吧！

 *  `runtime/pprof`：采集程序（`非 Server`）的运行数据进行分析
 *  `net/http/pprof`：采集 `HTTP Server` 的运行时数据进行分析

由于代码不在`http server`中，所以有关`net/http/pprof`的使用，以后再演示吧

##### 生成cpuprofile文件 #####

```bash
cd 到单元测试文件目录

go test -bench . -cpuprofile cpu.out
```

![2019-08-19-073612][]

执行性能测试完成后，会在当前文件夹生成两个文件

![2019-08-19-73613][]

这里主要关注cpu.out文件，该文件是一个二进制文件，可以使用golang内置的工具打开。

##### 使用go tool pporf 查看cpuprofile文件 #####

```bash
go tool pprof cpu.out
```

执行该命令后，会再次进入一个交互式控制台，输入相关指令，就能对cpuprofile文件做一些操作，  
比如：输入web，直接打开可视化界面，分析调用链

![2019-08-19-073614][]

第一次可能出现以上问题，按照提示，安装相关软件即可：`brew install Graphviz`

![2019-08-19-73617][]

分析调用链，颜色较深的部分为耗时占比较多的部分，  
通过分析，不难看出，主要耗时出现在了`func (node *Node) PrintValue()`函数中。

![2019-08-19-73619][]

这也比较好理解了，频繁的打印必然消耗IO嘛，找到问题了后续就慢慢优化吧！


[f2e0081d-247a-4651-acf3-4c6828162db4.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20190304/f2e0081d-247a-4651-acf3-4c6828162db4.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[c0a7f813-9345-4f89-b790-ff384b54e32f.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20190304/c0a7f813-9345-4f89-b790-ff384b54e32f.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[2019-08-19-073608]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20190304/1bba48f0-3f7a-4951-8c4c-8834437ab6fb.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[2019-08-19-073609]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20190304/6d3ab883-f1af-4177-b6c4-1de43d6acb1b.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[2019-08-19-073611]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20190304/1dfc09bc-8df8-4eba-9f2d-6b04650ec700.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[2019-08-19-073612]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20190304/3099ee0f-931f-4aad-8b3b-00a3f6a16250.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[2019-08-19-73613]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20190304/2d611c9a-60dc-48ab-a0cf-8e0f5ff0b79c.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[2019-08-19-073614]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20190304/6f104a49-9428-4733-8905-537ef5e3806f.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[2019-08-19-73617]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20190304/804f69e4-e127-4b4c-aa45-49afb392b402.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[2019-08-19-73619]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20190304/50fcbebe-2bd0-4aba-b428-9bae1ff05589.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg