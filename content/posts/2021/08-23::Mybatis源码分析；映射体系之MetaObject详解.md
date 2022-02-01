+++
title = "Mybatis源码分析；映射体系之MetaObject详解"
date = "2021-08-23 14:21:52"
url = "archives/1050"
tags = ["MyBatis"]
categories = ["后端","Mybatis源码阅读"]
featuredImage = "https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210923/bb73be20852d4c1392cecf5eb60fee7a.jpg"
+++

## 使用示例 ##

在处理SQL的参数映射以及结果集的映射时，Mybatis使用了大量的`MetaObject`对象，该对象也是Mybatis中设计得比较精妙的一个组件，主要用来处理JavaBean与参数以及结果集之间的相互映射，其核心逻辑即为反射。

在开始研究前，先来两个简单示例，感受一下`MetaObject`的强大之处。

```java
public class MetaObjectTest {

    @Data
    static class User {
        private String name;
        private UserGroup userGroup;
        private List<Tag> tags = new ArrayList<>();
    }

    @Data
    @NoArgsConstructor
    @AllArgsConstructor
    static class UserGroup {
        private String name;
    }

    @Data
    @NoArgsConstructor
    @AllArgsConstructor
    static class Tag {
        private String name;
    }


    @Test
    public void test1() {
        final User user = new User();
        // 使用MetaObject装饰对象
        final MetaObject metaObject = new Configuration().newMetaObject(user);
        metaObject.setValue("name", "张三");
        metaObject.setValue("userGroup.name", "管理组");

        // MetaObjectTest.User(name=张三, userGroup=MetaObjectTest.UserGroup(name=管理组), tagSet=[])
        System.out.println(user);
    }

    @Test
    public void test2() {
        final User user = new User();
        user.setName("李四");
        user.setUserGroup(new UserGroup("客服组"));

        final List<Tag> tags = new ArrayList<>();
        for (int i = 0; i < 3; i++)
            tags.add(new Tag("Tag-" + i));

        user.setTags(tags);

        final MetaObject metaObject = new Configuration().newMetaObject(user);
        // 李四
        System.out.println(metaObject.getValue("name"));
        // 客服组
        System.out.println(metaObject.getValue("userGroup.name"));
        // Tag-0
        System.out.println(metaObject.getValue("tags[0].name"));
    }
}
```

如单元测试所示，我们可以通过`MetaObject`对象装饰普通的`JavaBean`，来实现强大的类似于`JSONPATH`的功能，由于`Mybatis`是一个强大的`ORM`框架，势必会涉及到大量的反射处理，该工具类的出现，大大的简化了`Mybatis`框架内部操作属性的复杂度。

## MetaObject底层结构 ##

`MetaObject`相继依赖了`BeanWrapper`、`MetaClass`、`Reflector`，他们之间的关系如下图所示

![a4f29708-f384-4baa-b2e6-819f65222365.png][]

整体的依赖结构大致如以下伪代码：

```java
class MetaObject {
    ObjectWrapper objectWrapper; // 实现类有BaseWrapper，BeanWrapper，MapWrapper, CollectionWrapper
}

class BeanWrapper {
    MetaClass metaClass;
}

class MetaClass {
    Reflector reflector;
}
```

 *  **BeanWrapper**：与`MetaObject`类似，不同点是`BeanWrapper`只针对当前对象的属性进行操作，不支持子属性。
 *  **MetaClass**：类的反射功能支持，能获取完整类的属性，包括子属性类的属性。
 *  **Refector**：类的反射功能支持，只针对当前类的属性，不支持子属性。

## MetaObject获取属性流程 ##

为了便于分析完整的获取属性流程，咱们再来写一个稍微复杂一点的数据结构

```java
public class MetaObjectTest2 {
    @Test
    public void test() {
        final User user = buildUser();

        final MetaObject metaObject = new Configuration().newMetaObject(user);

        // 查找user对象中第一个tag的创建者姓名
        final Object value = metaObject.getValue("tags[0].creator.name");
        System.out.println(value);
    }


    @Data
    static class User {
        private String name;
        private List<Tag> tags = new ArrayList<>();
    }

    @Data
    @NoArgsConstructor
    @AllArgsConstructor
    static class Tag {
        private String name;
        private User creator;
    }

    static User buildUser() {
        final User user = new User();
        user.setName("张三");

        final List<Tag> tags = new ArrayList<>();
        for (int i = 0; i < 3; i++) {
            final User creator = new User();
            creator.setName("Creator-" + i);
            tags.add(new Tag("Tag-" + i, creator));
        }
        user.setTags(tags);
        return user;
    }
}
```

![96772b93-64c2-4e92-abd4-5bf172d2b746.png][]

首先，将传入的表达式经过属性分词器（`PropertyTokenizer`）进行分解。

```java
// 示例：tags[0].creator.name
public class PropertyTokenizer implements Iterator<PropertyTokenizer> {

  // 当前属性名: tags
  private String name;

  // 索引名：tags[0]
  private final String indexedName;

  // 索引（如果有）：0
  private String index;

  // 子属性：creator.name
  private final String children;

  public PropertyTokenizer(String fullname) {
    int delim = fullname.indexOf('.');
    if (delim > -1) {
      name = fullname.substring(0, delim);
      children = fullname.substring(delim + 1);
    } else {
      name = fullname;
      children = null;
    }
    indexedName = name;
    delim = name.indexOf('[');
    if (delim > -1) {
      index = name.substring(delim + 1, name.length() - 1);
      name = name.substring(0, delim);
    }
  }

  @Override
  public boolean hasNext() {
    // 判断是否有子属性
    return children != null;
  }

  @Override
  public PropertyTokenizer next() {
    // 继续解析子属性，再获得一个PropertyTokenizer对象
    return new PropertyTokenizer(children);
  }
}
```

再通过`PropertyTokenizer.hasNext()`判断是否有子属性，如果有，则调用`metaObjectForProperty()`方法继续处理子属性，反之则直接使用依赖的`objectWrapper`（这里用的是`BeanWrapper`）进行获取。

![82ac6f51-fd7e-4dc1-bfa3-e7689b123e42.png][]

在`metaObjectForProperty()`方法中，将子属性继续转换为`MetaObject`对象，并继续调用`metaObject.getValue()`方法

![ef782487-c309-49a1-bbd2-179f6609cbdb.png][]

最终。所有的子属性递归（类似）完成后，调用`objectWrapper.get()`来实现当前类底层的属性获取。那么接着往下看

![af8acc04-a1f7-44c3-8be6-bf0103331bd3.png][]

![2818a0b6-aeb2-4adf-98bd-eb42f8a8b5bf.png][]

通过`BeanWrapper`对象中依赖的`MetaClass`对象，获取到对应的`MethodInvoker`对象，最终实现`getName()`的反射调用。

![4deb7e13-2c7f-43a0-970f-0c50d60f5155.png][]

该示例涉及到多级嵌套，故而会执行多次`MethodInvoker.invoke()`，分别是`user.getTags()` \-> `tag.getCreator()`\-> `creator.getName()`;

最后，来个简单的流程图总结一下吧

![1b8cc5a1-a279-4d41-a3c4-6feaf0fa8169.png][]


[a4f29708-f384-4baa-b2e6-819f65222365.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210823/a4f29708-f384-4baa-b2e6-819f65222365.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[96772b93-64c2-4e92-abd4-5bf172d2b746.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210823/96772b93-64c2-4e92-abd4-5bf172d2b746.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[82ac6f51-fd7e-4dc1-bfa3-e7689b123e42.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210823/82ac6f51-fd7e-4dc1-bfa3-e7689b123e42.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[ef782487-c309-49a1-bbd2-179f6609cbdb.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210823/ef782487-c309-49a1-bbd2-179f6609cbdb.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[af8acc04-a1f7-44c3-8be6-bf0103331bd3.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210823/af8acc04-a1f7-44c3-8be6-bf0103331bd3.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[2818a0b6-aeb2-4adf-98bd-eb42f8a8b5bf.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210823/2818a0b6-aeb2-4adf-98bd-eb42f8a8b5bf.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[4deb7e13-2c7f-43a0-970f-0c50d60f5155.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210823/4deb7e13-2c7f-43a0-970f-0c50d60f5155.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[1b8cc5a1-a279-4d41-a3c4-6feaf0fa8169.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210823/1b8cc5a1-a279-4d41-a3c4-6feaf0fa8169.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg