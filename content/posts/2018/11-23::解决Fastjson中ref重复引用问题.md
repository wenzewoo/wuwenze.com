+++
title = "解决Fastjson中ref重复引用问题"
date = "2018-11-23 06:11:00"
url = "archives/111"
tags = ["Fastjson"]
categories = ["后端"]
+++

解决FastJson中"$ref重复引用"的问题，先来看一个例子吧：

```java
public static void main(String[] args) {
   UserGroup userGroup = new UserGroup().setName("UserGroup");

   User user = new User("User");
   for (int i = 0; i < 3; i++) {
       userGroup.addUser(user);
   }
   Console.log(JSON.toJSONString(userGroup));
}


@Data
@AllArgsConstructor
static class User {
   private String name;
}

@Data
@Accessors(chain = true)
static class UserGroup {
   private String name;
   private List<User> users = Lists.newArrayList();

   public UserGroup addUser(User user) {
       this.getUsers().add(user);
       return this;
   }
}
```

输出结果：

```json
{"name":"UserGroup","users":[{"name":"User"},{"$ref":"$.users[0]"},{"$ref":"$.users[0]"}]}
```

上面的现象就是将user对象的引用重复使用造成了重复引用问题，Fastjson默认开启引用检测将相同的对象写成引用的形式：

```bash
{"$ref": "$"} // 引用根对象
{"$ref":"@"} // 引用自己
{"$ref":".."} // 引用父对象
{"$ref":"../.."} // 引用父对象的父对象
{"$ref":"$.members[0].reportTo"} // 基于路径的引用
```

目前来说，前端还没有一个很好的办法来解析这样的JSON格式。

除了上面的重复引用外， 还衍生出了另外一个概念："循环引用"，下面来看下两者之间的区别吧：

 *  重复引用：指一个对象引用重复出现多次
 *  循环引用：对象A引用对象B，对象B引用对象A（这种情况一般是个雷区，轻易不要尝试的好，很容易引发StackOverflowError）

再来看一个循环引用的例子：

```java
public static void main(String[] args) {
    Order order = new Order().setName("Order");
    Item item = new Item().setName("Item");

    item.setOrder(order);
    order.setItem(item);

    Console.log(JSON.toJSONString(order));
    Console.log("----------------------------");
    Console.log(JSON.toJSONString(item));
}

@Data
@Accessors(chain = true)
static class Order {
    private String name;
    private Item item;
}

@Data
@Accessors(chain = true)
static class Item {
    private String name;
    private Order order;
}

// {"item":{"name":"Item","order":{"$ref":".."}},"name":"Order"}
// {"name":"Item","order":{"item":{"$ref":".."},"name":"Order"}}
```

### 解决方案 ###

 *  关闭FastJson引用检测机制（慎用，循环引用时可能导致`StackOverflowError`）


```java
JSON.toJSONString(obj, SerializerFeature.DisableCircularReferenceDetect)
```

 *  避免循环引用（某一方的引用字段不参与序列化：`@JSONField(serialize=false)`）
 *  避免一个对象引用被重复使用多次（使用拷贝的对象副本来完成JSON数据填充）

    
```java
public static void main(String[] args) {
    UserGroup userGroup = new UserGroup().setName("UserGroup");

    User user = new User("User");
    for (int i = 0; i < 3; i++) {
        User duplicateUser = new User();
        BeanUtil.copyProperties(user, duplicateUser);
        userGroup.addUser(duplicateUser);
    }
    Console.log(JSON.toJSONString(userGroup));
}
```