+++
title = "Golang实现有序遍历Map"
date = "2019-04-09 10:24:00"
url = "archives/546"
tags = ["Golang"]
categories = ["后端"]
+++

Go的`map`元素遍历顺序(使用`range`关键字)是随机的，而不是遵循元素的添加顺序，为解决这一问题，可使用下面的遍历方式。

```go
package main

import (
    "fmt"
    "sort"
)

func main() {
    map1 := map[int]string{
    0:	"java",
    1:	"golang",
    2:	"python",
    }

    var keys []int
    for key := range map1 {
        keys = append(keys, name)
    }
    sort.Ints(keys)
    for _, key := range keys {
        fmt.Printf("%st%dn", key, map1[key])
    }
}
```

可以看到，其关键思路就是先把所有Key拿出来排序，再按照排序后的进行遍历，主要用到了`sort`包的东西

关于sort还有些更个性化的用法，再看下面一个例子：

```go
package main

import (
    "fmt"
    "sort"
)

type Person struct {
    Name string
    Age  int16
}

type personSlice []Person

func (p personSlice) Len() int {
    return len(p)
}

func (p personSlice) Less(i, j int) bool {
    return p[i].Age > p[j].Age
}

func (p personSlice) Swap(i, j int) {
    p[i], p[j] = p[j], p[i]
}

func main() {
    people := personSlice{
        {Name: "张三", Age: 10},
        {Name: "王五", Age: 50},
        {Name: "李四", Age: 30},
    }
    sort.Sort(people)
    fmt.Println(people)
}

// [{王五 50} {李四 30} {张三 10}]
```

通过实现`go/src/sort/sort.Interface`接口，自行指定`struct`结构体的排序规则，这一点跟java有些类似。