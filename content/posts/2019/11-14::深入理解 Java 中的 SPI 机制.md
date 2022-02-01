+++
title = "深入理解 Java 中的 SPI 机制"
date = "2019-11-14 16:56:00"
url = "archives/661"
tags = ["Java"]
categories = ["后端"]
+++

### 概述 ###

`SPI`(Service Provider Interface)是一种比较流行的服务发现机制，其核心原理是通过在扫描`classpath:META-INF/services`文件夹下定义文件，来实现自动加载某个接口的实现类。

这种机制常为很多框架的扩展提供了便捷，比较常见的，如 Dubbo、JDBC 等。

### 小案例 ###

下面先通过一个小案例，来看看我们是如何利用这种机制的：

// 定义一个接口，该接口是项目的启动器

```java
package com.wuwenze.spi;

public interface Launcher {

    void run();
}
```

// 实现定义的 Launcher，该实现类用模拟使用 Jetty 容器来启动项目

```java
package com.wuwenze.spi.impl;

import com.wuwenze.spi.Launcher;

public class JettyLauncher implements Launcher {
    @Override
    public void run() {
        System.out.println("JettyLauncher is running...");
    }
}
```

// 再提供一个 Tomcat 的实现

```java
package com.wuwenze.spi.impl;

import com.wuwenze.spi.Launcher;

public class TomcatLauncher implements Launcher {
    @Override
    public void run() {
        System.out.println("TomcatLauncher is running...");
    }
}
```

// 最后一步，在`classpath:META-INF/services`文件夹下定义文件，文件名为接口的全名

com.wuwenze.spi.Launcher

```java
com.wuwenze.spi.impl.JettyLauncher
com.wuwenze.spi.impl.TomcatLauncher
```

![143649\_b423d88b\_955503][143649_b423d88b_955503]

// 然后我们就可以利用`ServiceLoader.load()`加载该接口的实现，进行测试了  
![144018\_f5997f15\_955503][144018_f5997f15_955503]

### ServiceLoader是如何实现的？ ###

带着疑问，跟一下ServiceLoader的源代码吧~

```java
public final class ServiceLoader<S>
    implements Iterable<S>
{

    // 配置实现类文件的路径，刚才已经用过了，按照官方的规范，这个是不能改路径的
    private static final String PREFIX = "META-INF/services/";

    // 表示正在加载的服务的类或接口
    private final Class<S> service;

    // 用于查找，加载和实例化提供程序的类加载器
    private final ClassLoader loader;

    // 创建ServiceLoader时采取的访问控制上下文
    private final AccessControlContext acc;

    // 已经缓存的实现类（按实例顺序）
    private LinkedHashMap<String,S> providers = new LinkedHashMap<>();

    // 内部类，用于查找实现类
    private LazyIterator lookupIterator;

    //...
}
```

类的成员，基本都看了，想必大概也猜到是如何实现的了，查找实现类和创建实现类的过程，都在LazyIterator完成。ServiceLoader本质上其实是个迭代器，当我们进行迭代操作时，实际上调用的都是LazyIterator的相应方法。

```java
public Iterator<S> iterator() {
        return new Iterator<S>() {

        Iterator<Map.Entry<String,S>> knownProviders
            = providers.entrySet().iterator();

        public boolean hasNext() {
            if (knownProviders.hasNext())
                return true;
            return lookupIterator.hasNext();
        }

        public S next() {
            if (knownProviders.hasNext())
                return knownProviders.next().getValue();
            return lookupIterator.next();
        }

        public void remove() {
            throw new UnsupportedOperationException();
        }

    };
}
```

这里，我们重点关注`lookupIterator.hasNext()`方法，然后会调用到hasNextService()：

```java
// 该方法用于获取实现类的类名
private boolean hasNextService() {
    if (nextName != null) {
        return true;
    }
    if (configs == null) {
        try {
            // META-INF/services/com.wuwenze.spi.Launcher
            String fullName = PREFIX + service.getName();

            // 将文件路径转成URL对象
            if (loader == null)
                configs = ClassLoader.getSystemResources(fullName);
            else
                configs = loader.getResources(fullName);
        } catch (IOException x) {
            fail(service, "Error locating configuration files", x);
        }
    }
    // 读取文件内容
    while ((pending == null) || !pending.hasNext()) {
        if (!configs.hasMoreElements()) {
            return false;
        }
        pending = parse(service, configs.nextElement());
    }
    // 拿到第一个实现类的类名
    nextName = pending.next();
    return true;
}
```

当进行迭代器的`.next()`操作时，这时就进行真正的对象实例化操作了，这时走到了nextService():

```java
private S nextService() {
    if (!hasNextService())
        throw new NoSuchElementException();
    String cn = nextName;
    nextName = null;
    Class<?> c = null;
    try {
        c = Class.forName(cn, false, loader);
    } catch (ClassNotFoundException x) {
        fail(service,
             "Provider " + cn + " not found");
    }
    if (!service.isAssignableFrom(c)) {
        fail(service,
             "Provider " + cn  + " not a subtype");
    }
    try {
        S p = service.cast(c.newInstance());
        providers.put(cn, p);
        return p;
    } catch (Throwable x) {
        fail(service,
             "Provider " + cn + " could not be instantiated",
             x);
    }
    throw new Error();          // This cannot happen
}
```

上面的代码就很明显了，通过反射`Class.forName()`轻松获得实例。

### 总结 ###

SPI底层实现实际上就是反射，Java 的反射特性真是太方便了，吹爆~


[143649_b423d88b_955503]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20191114/65a30051-4f70-4e8e-98db-428cd00701f8.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[144018_f5997f15_955503]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20191114/2d3d4894-13a4-42f7-9faf-6d9787e86244.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg