+++
title = "Java8 Stream Api使用详解"
date = "2018-08-07 06:32:00"
url = "archives/134"
tags = ["Java"]
categories = ["后端"]
+++

jdk8发布至今已有几年有余，是一个影响深远且具有革命意义的版本，目前jdk版本已直奔v11.0, 发展之迅速让人始料未及。本文在假设已有 `java8 lambda` 语法的基础下，通过几个示例，快速上手Stream 流处理相关的 API 使用。

### 什么是流操作 ###

流操作就是一条流水线，将元素放在流水线上一个个地进行处理: 

```java
List<User> list = Lists.newArrayList(user1,user2);
List<String> resultList =
    list.
    // 将集合转换成流对象
    stream()
    // 将List<User>遍历，将每个元素的name取出，组装成新的List<String>
    .map(User::getName)
    // 按照默认规则排序
    .sorted()
    // 只取20条数据
    .limit(20)
    // 收集流数据，组装成最终需要集合(List<String>)
    .collect(toList())
```

在以上的代码中，通过短短的几行代码，行云流水般的完成了一系列操作，这些操作在 jdk7 之前，是远远不能如此简单明了而高效的。

### 常用 API 概述 ###

> 在开始熟悉相关 API 之前，先做一些必要的准备工作，初始化一些数据。

```java
@Data
public class User {
    private String username;
    private Integer age;
}

List<User> list = new ArrayList<>();
list.add(new User("tom", 20));
list.add(new User("libai", 19));
list.add(new User("luxun", 55));
```

#### stream() ####

在开始操作之前，需要将一个普通的集合转换成流对象（Stream）

创建的方式很多，这里目前来说使用最直接也是常见的方式：

```java
list.stream();
```

#### filter(T -> boolean) ####

顾名思义，通过该函数将数据源筛选并保留lambda表达式中为 true的值。

```java
Stream<?> stream = list.stream()
    .filter(user -> user.getAge() == 20)
```

#### sorted() / sorted((T, T) -> int) ####

如果流中的元素的类实现了 Comparable 接口，即有自己的排序规则，那么可以直接调用 sorted() 方法对元素进行排序，反之, 则需要调用 sorted((T, T) -> int) 自己实现 Comparator 接口。

```java
Stream<?> stream = list.stream()
       .sorted((u1, u2) -> u1.getAge() - u2.getAge())
```

以上形式的代码还可以继续简写，这属于lambda相关的知识：

```java
Stream<?> stream = list.stream()   .sorted(Comparator.comparingInt(User::getAge))
```

#### limit(long n) ####

返回前 n 个元素

```java
Stream<?> stream = list.stream().limit(20);
```

#### skip(long n) ####

跳过前 n 个元素

```java
Stream<?> stream = list.stream().skip(20);
```

#### map(T -> R) ####

将流中的每一个元素 T 映射为 R（类似类型转换）

```java
Stream<?> stream = list.stream().map(User::getName)
```

#### anyMatch(T -> boolean) ####

流中是否有一个元素匹配给定的 T -> boolean 条件

#### allMatch(T -> boolean) ####

流中是否所有元素都匹配给定的 T -> boolean 条件

#### noneMatch(T -> boolean) ####

流中是否没有元素匹配给定的 T -> boolean 条件

#### findAny() 和 findFirst() ####

findAny()：找到其中一个元素 （使用 stream() 时找到的是第一个元素；使用 parallelStream() 并行时找到的是其中一个元素）

findFirst()：找到第一个元素

值得注意的是，这两个方法返回的是一个 Optional 对象，它是一个容器类，能代表一个值存在或不存在，这个后面会讲到

#### reduce((T, T) -> T) 和 reduce(T, (T, T) -> T) ####

用于组合流中的元素，如求和，求积，求最大值等

```java
// 计算年龄总和：
int sum = list.stream().map(User::getAge).reduce(0, (a, b) -> a + b);
// 与之相同:
int sum = list.stream().map(User::getAge).reduce(0, Integer::sum);
```

#### count() ####

返回流中元素个数，结果为 long 类型

#### collect() ####

收集方法，我们很常用的是 collect(toList())，当然还有 collect(toSet()) 等...

#### forEach() ####

迭代器

Stream API 相当的强大，本文只是列举一些皮毛，起个抛砖引玉的效果。