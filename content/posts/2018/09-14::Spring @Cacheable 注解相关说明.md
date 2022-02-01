+++
title = "Spring @Cacheable 注解相关说明"
date = "2018-09-14 06:46:00"
url = "archives/384"
tags = ["Spring"]
categories = ["后端"]
+++

关于`@Cacheable`注解的作用不做过多说明，文本主要针对该注解的key自定义策略规则提供一些示例。

### @Cacheable ###

| 属性名       | 必填? | 描述                             |
|-----------|-----|--------------------------------|
| value     | 必填  | 缓存的命名空间                        |
| key       | 可选  | 指定一个唯一的key（在缓存命名空间中），使用SpEL表达式 |
| condition | 可选  | 限定条件，哪种情况使用缓存，使用SpEL表达式        |
| unless    | 可选  | 限定条件，哪种情况下不使用缓存，使用SpEL表达式      |


### key ###

默认情况下，key属性可以不填，Spring会按照默认策略去生成缓存相关的key（这里不细究了）

该属性在自定义时，通常使用`SpEL表达式`来声明，以下是一些常见的例子：

通过方法参数作为key：

```java
@Cacheable(value = "findByUserIdCached", key = "#userId")
public User findByUserIdCached(Integer userId) {
    return userDao.findByUserId(userId);
}

// 还有一种写法：#p0 表示第一个参数
@Cacheable(value = "findByUserIdCached", key = "#p0")
public User findByUserIdCached(Integer userId) {
    return userDao.findByUserId(userId);
}

@Cacheable(value = "findByUserIdCached", key = "#user.id")
// @Cacheable(value = "findByUserIdCached", key = "#p0.id")
public User findByUserIdCached(User user) {
    return userDao.findByUserId(user.getId());
}

// 多个参数拼接
@Cacheable(value = "findByUsernameAndTypeCached", key = "#username + '_' + ##type")
// @Cacheable(value = "findByUsernameAndTypeCached", key = "#p0 + '_' + ##p1")
public User findByUsernameAndTypeCached(String username, Integer type) {
    return null;
}

// 复杂参数（数组）
@Cacheable(value = "findByUserIdsCached", key = "#userIds[0] + ''")
// @Cacheable(value = "findByUserIdsCached", key = "#p0[0] + ''")
public User findByUserIdsCached(Integer[] userIds) {
    return null;
}
```

除此之外，Spring还提供了一个root对象：。

| 属性          | 描述               | 示例                   |
|-------------|------------------|----------------------|
| methodName  | 当前方法名            | #root.methodName     |
| method      | 当前方法             | #root.method.name    |
| target      | 当前被调用的对象         | #root.target         |
| targetClass | 当前被调用的对象的class   | #root.targetClass    |
| args        | 当前方法参数数组         | #root.args[0]        |
| caches      | 当前被调用的方法使用的Cache | #root.caches[0].name |


```java
@Cacheable(value = "findByUserIdCached", key = "#root.methodName + '_' + ##user.id")
public User findByUserIdCached(User user) {
    return userDao.findByUserId(user.getId());
}

// 当然，也可以通过root对象调用当前类的public方法
class TestService {

    @Cacheable(value = "findByUserIdCached", key = "#root.target.getName() + '_' + ##user.id")
    public User findByUserIdCached(User user) {
        return userDao.findByUserId(user.getId());
    }

    public String getName() {
        return "test";
    }
}
```

### condition && unless ###

此外，condition以及unless都是可以通过配置SpEL表达式来完成的，最终需要一个条件表达式，如：

```java
// 只有当type为偶数时，才使用缓存。
@Cacheable(value = "findByUsernameAndTypeCached", key = "#username + '_' + ##type", condition = "#type % 2 == 0")
public User findByUsernameAndTypeCached(String username, Integer type) {
    return null;
}

// 同理，只有当type为偶数时，才不使用缓存
@Cacheable(value = "findByUsernameAndTypeCached", key = "#username + '_' + ##type", unless = "#type % 2 == 0")
public User findByUsernameAndTypeCached(String username, Integer type) {
    return null;
}
```