+++
title = "造个轮子::实现一个极简的Goroutine Pool"
date = "2020-01-10 16:59:00"
url = "archives/670"
tags = ["Golang"]
categories = ["后端"]
+++

### 概述 ###

Go语言的协程(Goroutine)是一种相对线程而言更廉价的方式，虽然是轻量级的，但Goroutine太多仍会导致调度性能下降、GC 频繁、内存暴涨, 引发一系列问题。在面临这样的场景时, 限制 Goroutine的数量、重用显然很有价值。

### 解决方案 ###

要解决这个问题，首先要考虑的是以下几点：

 *  Goroutine的数量如何限制
 *  Goroutine如何重用，不要频繁创建
 *  Goroutine如何执行，管理等

#### 关于限制数量和重用 ####

说到限制和重用, 那么最先想到的就是池化（如 JDBC池、线程池等都是最佳实践）所以我们也来造一个轮子，来实现一个轻量级的协程池（暂且不管市面上已有很成熟的项目，本文只是为了技术研究）

#### 任务如何执行 ####

在使用原生goroutine的场景中, 运行一个任务直接启动一个goroutine来运行, 在池化的场景而言, 任务也是要在goroutine中执行, 但是任务需要任务池来放入 goroutine。

##### 使用生产者消费者模型 #####

在一般连接池中, 连接在使用时从池中取出, 用完后放入池中即可。  
但是对于goroutine而言, 其通过语言关键字启动, 无法像其他语言那么方便了。那么如何让goroutine可以执行任务, 且执行后可以重新用来执行其它任务呢？这里就需要使用生产者消费者模型了:

生产者 --(生产任务)--> 队列 --(消费任务)--> 消费者

用来执行任务的goroutine可以作为消费者, 操作任务池的goroutine作为生产者, 而队列则可以使用buffer channel。

### 具体实现 ###

#### 定义 Task ####

```go
type Task struct {
    // 任务名
    Name string
    // 任务回调函数
    Handler func(v ...interface{})
    // 任务回调函数参数（如果有）
    Params []interface{}
}
```

#### 定义 TaskPool ####

```go
type State int32

const (
    StateRunning State = iota
    StateStopped
)

type TaskPool struct {
    // 最大容量
    capacity int32
    // 协程池状态
    state State
    // 运行中的任务个数
    runningTasks int32
    // 任务管道
    taskChannel chan *Task
}
```

##### 构造函数 #####

```go
func NewTaskPool(capacity int32) (*TaskPool, error) {
    if capacity <= 0 {
        return nil, errors.New("capacity less than 0")
    }
    return &TaskPool{
        capacity:    capacity,
        state:       StateRunning,
        taskChannel: make(chan *Task, capacity),
    }, nil
}
```

#### 启动 worker ####

```go
func (tp *TaskPool) run() {
    // 运行中的数量+1
    atomic.AddInt32(&tp.runningTasks, 1)

    go func() {
        defer func() {
            // 运行中的任务-1
            atomic.AddInt32(&tp.runningTasks, -1)

            // 错误收集（暂时只打印）
            if err := recover(); err != nil {
                log.Printf("Worker error: %sn", err)
            }
        }()

        for {
            select {
            case task, ok := <-tp.taskChannel:
                if ok {
                    log.Printf("Task[%s] start execution...n", task.Name)
                    task.Handler(task.Params...)
                }
            }
        }
    }()
}
```

上述代码中, runningTasks的加减运算使用sync.atomic包来保证其自增操作是原子的。

#### 生产任务 ####

```go
func (tp *TaskPool) AddTask(task *Task) error {
    if tp.state == StateStopped {
        return errors.New("task pool is closed")
    }
    runningTasks := atomic.LoadInt32(&tp.runningTasks)
    capacity := atomic.LoadInt32(&tp.capacity)

        // 如果当前运行的任务小于协程池最大限制，则通知消费者开始消费
    if runningTasks < capacity {
        tp.run() // 消费者启动
    }
    // 生产者将 task 放入管道
    tp.taskChannel <- task
    return nil
}
```

#### 安全（优雅）关闭 ####

```go
func (tp *TaskPool) Close() {
    tp.state = StateStopped
    for { // 一直阻塞，直到协程池中的所有 Task 被消费完毕，再关闭管道
        if len(tp.taskChannel) <= 0 {
            close(tp.taskChannel)
            return
        }
    }
}
```

### 使用 ###

```go
func TestTaskPool(t *testing.T) {
    // 新建协程池
    taskPool, err := NewTaskPool(10)
    if err != nil {
        panic(err)
    }

    // 提交 100 个任务，等待执行完成
    waitGroup := &sync.WaitGroup{}
    for i := 0; i < 100; i++ {
        waitGroup.Add(1)

        _ = taskPool.AddTask(&Task{
            Name: fmt.Sprintf("Demo-%d", i),
            Handler: func(v ...interface{}) {
                defer waitGroup.Done()

                fmt.Printf("Hello, %s n", v[0])
            },
            Params: []interface{}{fmt.Sprintf("name-%d", i)},
        })
    }

    waitGroup.Wait()
    taskPool.Close() // 安全关闭协程池
}
```

### TODO ###

 *  协程的复用（在实现中没有做到，目前来说只做到了限流）
 *  性能测试（对比原生Goroutine性能如何，暂且未知，后续可以着重测试一下）

以上就是协程池的极简封装了，当然就目前来说，只是简单按照原理实现了一遍，可能还有很多细节需要完善的，这里就不再继续下去了。