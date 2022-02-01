+++
title = "Golang实现HTTP请求限流"
date = "2019-04-29 16:17:00"
url = "archives/564"
tags = ["Golang","令牌桶"]
categories = ["后端"]
+++

在高并发应用场景中，为保证业务高峰期系统的稳定性或抵御CC攻击，最有效的方案为（缓存、降级、限流）  
本文以限流为例，在Golang中示例如何通过中间件实现httpserver限流。

### 依赖 ###

首先安装一个基本的限流算法包，然后为了方便，直接使用gin作为httprouter（后面会用到它的中间件）

```bash
$ go get -u golang.org/x/time/rate

$ go get -u github.com/gin-gonic/gin
```

### 全局限流 ###

定义全局限流中间件：`/middleware/global_limiter.go`

```go
package middleware

import (
    "github.com/gin-gonic/gin"
    "golang.org/x/time/rate"
    "net/http"
)

// 初始化一个限流器，最大请求上限5个
var limiter = rate.NewLimiter(2, 5)
func GlobalLimiter() gin.HandlerFunc {
    return func(ctx *gin.Context) {
        if limiter.Allow() == false {
            ctx.JSON(http.StatusTooManyRequests, gin.H{"status": "Too many Requests."})
            ctx.Abort()
            return
        }
        ctx.Next()
    }
}
```

使用全局限流器以及测试：`main.go`

```go
package main

import (
    "github.com/gin-gonic/gin"
    "golimit/middleware"
    "net/http"
)

func main() {
    app := gin.Default()
    app.Use(middleware.GlobalLimiter())

    app.GET("/say", func(ctx *gin.Context) {
        ctx.JSON(http.StatusOK, gin.H{"status": "ok"})
    })

    if err := app.Run(":9087"); err != nil {
        panic(err)
    }
}
```

简单测试：

```go
package main

import (
    "io/ioutil"
    "net/http"
    "testing"
)

func TestLimiter(t *testing.T)  {
    for i := 0;  i < 6; i++  {
        resp, err := http.Get("http://localhost:9087/say")
        if err != nil {
            t.Errorf("fetch error, %s", err.Error())
            return
        }
        defer resp.Body.Close()

        bytes, _ := ioutil.ReadAll(resp.Body)
        t.Logf("fetch success, %s", string(bytes))
    }
}
```

测试结果：

```bash
=== RUN   TestLimiter
--- PASS: TestLimiter (0.01s)
    main_test.go:19: fetch success, {"status":"ok"}
    main_test.go:19: fetch success, {"status":"ok"}
    main_test.go:19: fetch success, {"status":"ok"}
    main_test.go:19: fetch success, {"status":"ok"}
    main_test.go:19: fetch success, {"status":"ok"}
    main_test.go:19: fetch success, {"status":"Too many Requests."}
PASS

Process finished with exit code 0
```

### 粒度更细的IP限流 ###

一般来说，全局限流还是太过于粗暴了，我们可以将粒度更细一点，比如请求IP、请求用户等等

定义IP限流中间件：`/middleware/ip_limiter.go`

```go
package middleware

import (
    "github.com/gin-gonic/gin"
    "net/http"
    "sync"
    "time"

    "golang.org/x/time/rate"
)

type visitor struct {
    limiter  *rate.Limiter
    lastSeen time.Time
}

var visitors = make(map[string]*visitor)
var mtx sync.Mutex

func init() {
    go cleanupVisitors()
}

func addVisitor(ip string) *rate.Limiter {
    limiter := rate.NewLimiter(2, 5)
    mtx.Lock()
    visitors[ip] = &visitor{limiter, time.Now()}
    mtx.Unlock()
    return limiter
}

func getVisitor(ip string) *rate.Limiter {
    mtx.Lock()
    v, exists := visitors[ip]
    if !exists {
        mtx.Unlock()
        return addVisitor(ip)
    }

    v.lastSeen = time.Now()
    mtx.Unlock()
    return v.limiter
}

func cleanupVisitors() {
    for {
        time.Sleep(time.Minute)
        mtx.Lock()
        for ip, v := range visitors {
            if time.Now().Sub(v.lastSeen) > 3*time.Minute {
                delete(visitors, ip)
            }
        }
        mtx.Unlock()
    }
}
func IPLimiter() gin.HandlerFunc {
    return func(ctx *gin.Context) {
        clientIp := ctx.ClientIP()

        limiter := getVisitor(ctx.ClientIP())
        if limiter.Allow() == false {
            ctx.JSON(http.StatusTooManyRequests, gin.H{"status": "Too many Requests. IP=" + clientIp})
            ctx.Abort()
            return
        }
        ctx.Next()
    }
}
```

使用限流中间件：

```go
func main() {
    app := gin.Default()
    //app.Use(middleware.GlobalLimiter())
    app.Use(middleware.IPLimiter())

    app.GET("/say", func(ctx *gin.Context) {
        ctx.JSON(http.StatusOK, gin.H{"status": "ok"})
    })

    if err := app.Run(":9087"); err != nil {
        panic(err)
    }
}
```

当然，并不是所有接口都需要限流，你也可以这样：

```go
func main() {
    app := gin.Default()
    //app.Use(middleware.GlobalLimiter())
    //app.Use(middleware.IPLimiter())

  // 只有对外开放的公开接口需要限流。
    group := app.Group("/api/v1", middleware.IPLimiter())
    {
        group.GET("/say", func(ctx *gin.Context) {
            ctx.JSON(http.StatusOK, gin.H{"status": "ok"})
        })
    }
    
    app.GET("/test", func(ctx *gin.Context) {
        ctx.String(http.StatusOK, "ok")
    })

    if err := app.Run(":9087"); err != nil {
        panic(err)
    }
}
```

当然，实际应用场景中可能考虑得要多一点，但是大致实现就是这样子。