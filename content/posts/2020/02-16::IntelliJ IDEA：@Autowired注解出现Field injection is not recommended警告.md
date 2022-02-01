+++
title = "IntelliJ IDEA：@Autowired注解出现Field injection is not recommended警告"
date = "2020-02-16 17:04:00"
url = "archives/681"
tags = ["Spring","Intellij IDEA","Java"]
categories = ["后端"]
+++

在开发的时候，突然发现新版的IDEA在`@Autowired`注解上出现一个大大的警告（Field injection is not recommended）

```java
@RestController
public class UserRestController {
    @Autowired
    private UserService userService;
}
```

![183340\_77f5e60b\_955503][183340_77f5e60b_955503]

这谁忍得了？以我多年的使用Spring的经验来看，这段代码使用@Autowired来自动装配肯定没有问题，同时也一直是这么干的，虽然有人曾说，程序员重来不管警告，但是对一个重度代码洁癖患者而言，是绝对不允许一个不明不白的警告出现在这里的！

### 为什么出现这段警告？ ###

借助着IDEA的强大，我们来看看这段警告是究竟为什么

![190127\_a5b68d74\_955503][190127_a5b68d74_955503]  
翻译过来的意思为：

> 不建议使用字段注入  
> Spring团队建议：“在bean中始终使用基于构造函数的依赖注入。总是对强制依赖项使用断言”。

意思很明显了，只是一个建议，Spring团队为什么要如此建议呢？我们一向的做法都是不太规范的吗？

#### 干掉此警告 ####

在研究Spring团队为什么不建议这样做之前，我们先尝试干掉这段警告

 *  直接屏蔽掉该警告，眼不见心不烦（不建议）

![191254\_6bdfed62\_955503][191254_6bdfed62_955503]  
警告虽然消失了，但是这种掩耳盗铃之事，还是少干为妙。

 *  使用IDEA的自动修复建议

![191527\_3a96969e\_955503][191527_3a96969e_955503]

如图中所示，IDEA自动将字段注入修改为了构造函数注入（显然，这是Spring团队推荐的做法）

```java
@RestController
public class UserRestController {
    private final UserService userService;
    public UserRestController(UserService userService) {
        this.userService = userService;
    }
}
```

代码量无疑增加了不少，不能忍，这时候我们的`Lombok`将派出用场了：

```java
@RestController
@RequiredArgsConstructor
public class UserRestController {
    private final UserService userService;
}
```

`@RequiredArgsConstructor`注解的意思为：将没被初始化过的final的变量创建构造方法

#### Spring自动装配的所有姿势 ####

在研究字段注入的问题之前，我们来回顾一下Spring中都有哪些自动装配姿势

基于Field注入

```java
@Autowired
private UserService userService;
```

基于Setter注入

```java
private UserService userService;
public void setUserService(UserService userService) {
    this.userService = userService;
}
```

基于Constructor注入

```java
private final UserService userService;
public UserRestController(UserService userService) {
    this.userService = userService;
}
```

显然，第一种是我们目前日常开发中使用得最多，但是居然是官方唯一不建议的做法，这很难受。

#### 基于Field注入可能带来的风险 ####

既然不是建议的做法，那么一定有其缘由，带着这个疑问，我翻阅了一些资料，得到了答案，先来看下面这段代码：

```java
@RestController
public class UserRestController {
    @Autowired
    private UserService userService;
    private UserEntity userEntity;
    public UserRestController() {
        this.userEntity = userService.findById(1);
    }
}
```

暂且不管这段代码的使用场景，合理性，试想一下，这段代码能运行成功吗？答案显然是不能的

```bash
Exception in thread "main" org.springframework.beans.factory.BeanCreationException: Error creating bean with name '...' defined in file [....class]: 
  Instantiation of bean failed; nested exception is org.springframework.beans.BeanInstantiationException: Failed to instantiate [...]: Constructor threw exception; 
  nested exception is java.lang.NullPointerException
```

这是为何？仔细观察代码，不难看出，这段代码违背了初始化类的规则：

`静态变量或静态语句块` –> `实例变量或初始化语句块` –> `构造方法` -> `@Autowired`

果然，基于Field注入是有风险且在编译期间不容易被发现的，这就是为什么Spring团队不建议的原因了。

#### 总结 ####

既然Spring团队如此建议了，那么以后的开发中，则需要多多注意啊，一份严谨的且符合规范的代码，一直是我们的毕生追求。


[183340_77f5e60b_955503]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20200216/1affc626-9b2f-4124-b8be-a31910109235.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[190127_a5b68d74_955503]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20200216/79356352-78c0-4605-9ce8-a10ae062a764.gif?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[191254_6bdfed62_955503]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20200216/feb56067-9a5c-4a78-9e8f-a5f091b86d25.gif?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[191527_3a96969e_955503]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20200216/9de15558-9697-43cd-afe8-906aeb0c93f7.gif?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg