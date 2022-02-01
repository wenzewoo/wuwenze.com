+++
title = "GraphQL::为API而生的查询语言"
date = "2019-04-18 16:04:00"
url = "archives/551"
tags = ["GraphQL"]
categories = ["未分类"]
+++

## 概述 ##

GraphQL是Facebook开源的API查询语言，类似于数据库中的SQL。作为比较，RESTful API依赖于后端隐式的被动的数据约定，GraphQL更加显式，在获取数据和更新数据时更加主动，所见即所得。  
![2019-08-19-073546][]  
详见官网：[http://graphql.cn/][http_graphql.cn]

## 谁在使用？ ##

Github早就开放了一套基于GraphQL的api，可以试试。[https://developer.github.com/v4/][https_developer.github.com_v4]

## 与传统RESTful API对比 ##

#### RESTful的一些不足 ####

 *  扩展性，为了满足各种场景的业务，单个RESTful接口返回数据越来越臃肿，而前端又不一定用得上。
 *  某个前端展现，实际需要调用多个独立的RESTful API才能获取到足够的数据
 *  暴露的端点越来越多，越来越难维护

#### GraphQL的优点 ####

 *  **请求你的数据不多不少**：GraphQL查询总是能准确获得你想要的数据，不多不少，所以返回的结果是可预测的。
 *  **获取多个资源只用一个请求**：GraphQL查询不仅能够获得资源的属性，还能沿着资源间进一步查询，所以GraphQL可以通过一次请求就获取你应用所需的所有数据。
 *  **描述所有的可能类型系统**：GraphQL API基于类型和字段的方式进行组成，使用类型来保证应用只请求可能的类型，同时提供了清晰的辅助性错误信息。
 *  **使用你现有的数据和代码**：GraphQL让你的整个应用共享一套API，通过GraphQL API能够更好的利用你的现有数据和代码。GraphQL 引擎已经有多种语言实现，GraphQL不限于某一特定数据库，可以使用已经存在的数据、代码、甚至可以连接第三方的APIs。
 *  **API 演进无需划分版本**： 给GraphQL API添加字段和类型而无需影响现有查询。老旧字段可以废弃，从工具中隐藏。

## 基于Golang的实现 ##

GraphQL已经支持市面上的大部分语言，并都提供了相关的SDK，可以在这个页面看到：[http://graphql.cn/code/][http_graphql.cn_code]

### graphql-go ###

```go
go get github.com/graphql-go/graphql
go get github.com/gin-gonic/gin

module github.com/wuwz/graphqldemo

require (
    github.com/gin-gonic/gin v1.3.0 // indirect
    github.com/graphql-go/graphql v0.7.8 // indirect
)
```

方便起见，这里我们用了gin来做路由。

### struct ###

```go
package model

import (
    "errors"
    "fmt"
    "math/rand"
)

type User struct {
    Id   int    `json:"id"`
    Name string `json:"name"`
    Age  int    `json:"age"`
}

// 初始化一些数据，这里就不到数据库去查询了
var users []User

func init() {
    for i := 0; i < 100; i++ {
        users = append(users, User{
            Id:   i,
            Name: fmt.Sprintf("name_%d", i),
            Age:  rand.Intn(40),
        })
    }
}

// 定义查询方法
func (*User) FindByID(id int) (*User, error) {
    for _, v := range users {
        if v.Id == id {
            return &v, nil
        }
    }
    return nil, errors.New(fmt.Sprintf("UserID=%d NotFound.", id))
}

func (*User) ListAll() (*[]User, error) {
    return &users, nil
}
```

### schema ###

```go
// schema/schema.go
package schema

import "github.com/graphql-go/graphql"

// 定义查询节点
var rootQuery = graphql.NewObject(graphql.ObjectConfig{
    Name:        "RootQuery",
    Description: "使用GraphQL提供的查询API",
    Fields: graphql.Fields{
        "findUserByID": &findUserByID, // 参考schema/user.go
        "listAll":      &listAll,
    },
})

// 定义Schema用于http handler处理
var Schema, _ = graphql.NewSchema(graphql.SchemaConfig{
    Query:    rootQuery,
    Mutation: nil, // 需要通过GraphQL更新数据，可以定义Mutation
})
```

### user-schema ###

```go
package schema

import (
    "github.com/graphql-go/graphql"
    "github.com/wuwz/graphqldemo/model"
)

// 定义查询对象的字段，支持嵌套
var userType = graphql.NewObject(graphql.ObjectConfig{
    Name:        "User",
    Description: "用户信息模型",
    Fields: graphql.Fields{
        "id": &graphql.Field{
            Type: graphql.Int,
        },
        "name": &graphql.Field{
            Type: graphql.String,
        },
        "age": &graphql.Field{
            Type: graphql.Int,
        },
    },
})

// 处理查询请求
var findUserByID = graphql.Field{
    Name:        "findUserByID",
    Description: "根据用户ID查询用户信息",
    Type:        userType,
    // Args是定义在GraphQL查询中支持的查询字段
    Args: graphql.FieldConfigArgument{
        "id": &graphql.ArgumentConfig{
            Type: graphql.Int,
        },
    },
    // Resolve是一个处理请求的函数，具体处理逻辑可在此进行
    Resolve: func(p graphql.ResolveParams) (result interface{}, err error) {
        // Args里面定义的字段在p.Args里面，对应的取出来
        id, _ := p.Args["id"].(int)
        return (&model.User{}).FindByID(id)
    },
}

var listAll = graphql.Field{
    Name:        "listAll",
    Description: "获取用户列表",
    Type:        graphql.NewList(userType),
    Resolve: func(p graphql.ResolveParams) (result interface{}, err error) {
        return (&model.User{}).ListAll()
    },
}
```

### router ###

```go
package main

import (
    "github.com/gin-gonic/gin"
    "github.com/graphql-go/handler"
    "github.com/wuwz/graphqldemo/schema"
    "log"
)

func main() {
    app := gin.Default()
    // 暴露的GraphQL端点
    app.Any("/graphql", GraphqlHandler())

    if err := app.Run(":10078"); err != nil {
        log.Panicf("run error, %s", err.Error())
    }
}

// 使用GraphqlHandler接管gin路由
func GraphqlHandler() gin.HandlerFunc {
    return func(ctx *gin.Context) {
        graphqlHandler := handler.New(&handler.Config{
            Schema:   &schema.Schema, // schema/schema.go
            Pretty:   true,
            GraphiQL: true, // 提供WEB UI查询界面
        })
        graphqlHandler.ServeHTTP(ctx.Writer, ctx.Request)
    }
}
```

浏览器访问：[http://localhost:10078/graphql][http_localhost_10078_graphql]

![2019-08-19-073547][]

还有更多好玩的，以后再说吧。


[2019-08-19-073546]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20190418/645d2c25-6b96-4773-adeb-6fd210154b9d.gif?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[http_graphql.cn]: http://graphql.cn/
[https_developer.github.com_v4]: https://developer.github.com/v4/
[http_graphql.cn_code]: http://graphql.cn/code/
[http_localhost_10078_graphql]: http://localhost:10078/graphql
[2019-08-19-073547]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20190418/8c05f431-279f-42fe-a56b-d0d9a7d0ac9b.gif?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg