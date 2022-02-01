+++
title = "Kotlin 单例模式详解"
date = "2019-07-08 16:43:00"
url = "archives/630"
tags = ["Kotlin"]
categories = ["后端"]
+++

单例模式很熟悉了，在Java中有各种创建姿势，算是比较麻烦的，这里就不再赘述了。

### Object ###

那么如何在kotlin中实现单例模式呢？请看代码

```kotlin
object JsonObjectMapper {
}
```

仅需简单的把`class`关键字替换为`object`就完成了！

```kotlin
// Kotlin 调用
JsonObjectMapper

// Java调用
JsonObjectMapper.INSTANCE
```

究竟Kotlin封装了什么细节？让我们一探究竟吧：

通过`Tools` > `Kotlin` > `Show Kotlin Bytecode` 显示字节码，看看翻译后的Java代码  
![2019-08-19-073703][]

![2019-08-19-073706][]

翻译成Java后，发现就是一个最简单的`饿汉式单例`而已。

### Companion Object ###

##### 懒汉式 #####

```kotlin
class SingletonDemo private constructor() {
    companion object {
        private var instance: SingletonDemo? = null
            get() {
                if (field == null) {
                    field = SingletonDemo()
                }
                return field
            }
        fun get(): SingletonDemo{
            return instance!!
        }
    }
}
```

##### 线程安全的懒汉式 #####

```kotlin
class SingletonDemo private constructor() {
    companion object {
        private var instance: SingletonDemo? = null
            get() {
                if (field == null) {
                    field = SingletonDemo()
                }
                return field
            }
        @Synchronized
        fun get(): SingletonDemo{
            return instance!!
        }
    }
}
```

##### 双重校验锁式（Double Check) #####

```kotlin
class SingletonDemo private constructor() { 
    companion object { 
        val instance: SingletonDemo by lazy(mode = LazyThreadSafetyMode.SYNCHRONIZED) { 
            SingletonDemo() 
        }
    }
}
```

##### 静态内部类式 #####

```kotlin
class SingletonDemo private constructor() {
    companion object {
        val instance = SingletonHolder.holder
    }
    private object SingletonHolder {
        val holder = SingletonDemo()
    }
}
```


[2019-08-19-073703]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20190708/313da8bb-7197-4efe-a6d3-a0ed707f9648.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[2019-08-19-073706]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20190708/71ea6b68-6c84-46bc-a16b-b69efe55b0c6.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg