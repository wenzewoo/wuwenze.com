+++
title = "当synchronized关键字作用在字符串上的一些问题"
date = "2019-11-21 16:58:00"
url = "archives/667"
tags = ["Java","并发编程"]
categories = ["后端"]
+++

## 概述 ##

现有需求如下，每个用户(租户ID:用户ID)执行某个操作时，对其进行计数，在考虑高并发的情况下，不会产生计数错误问题。

## 实现 ##

看到需求后，脑中迅速想好了一个简单思路：  
将用户(租户ID:用户ID)作为 KEY, 在执行查询和更新时，利用关键字`synchronized`锁之即可，下面是测试代码：

```java
public class Main {
    private static final Map<String, Integer> COUNT_MAP = Maps.newHashMap();

    public static void main(String[] args) {
        final List<Thread> threads = Lists.newArrayList();

        // 模拟 3 个用户
        for (int i = 0; i < 3; i++) {

            // 每个用户并发 100 次操作计数器
            for (int j = 0; j < 100; j++) {
                int userId = i;
                Thread thread = new Thread(() -> {
                    final String key = "providerId:" + userId;

                    int count = get(key);
                    set(key, count + 1);
                });
                threads.add(thread);
                thread.start();
            }
        }


        threads.forEach(thread -> {
            try {
                thread.join();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        });
        System.out.println(COUNT_MAP);
    }

    static Integer get(String key) {
        synchronized (key) {
            Integer count = COUNT_MAP.get(key);
            return null == count ? 0 : count;
        }
    }

    static void set(String key, Integer count) {
        synchronized (key) {
            COUNT_MAP.put(key, count);
        }
    }
}
```

然而运行了几次后，发现结果差强人意，难道是`synchronized`关键字没有起作用？

```bash
{providerId:2=98, providerId:1=97, providerId:0=98}
{providerId:0=100, providerId:1=99, providerId:2=100}
{providerId:2=100, providerId:0=100, providerId:1=99}
{providerId:0=100, providerId:1=100, providerId:2=100} // 最后一次总算是成功了
```

## 问题分析 ##

回头看看代码，不难发现，问题出现在`synchronized (key)` 这行代码上， key 在这里是一个字符串，每次构建 key 时，其实都是一个全新的对象，这当然锁不住啦！

## 解决方案 ##

对以上代码进行稍加改造，就可以解决我们的问题：

```java
static Integer get(String key) {
    synchronized (key.intern()) {
        Integer count = COUNT_MAP.get(key);
        return null == count ? 0 : count;
    }
}

static void set(String key, Integer count) {
    synchronized (key.intern()) {
        COUNT_MAP.put(key, count);
    }
}
```

这是为何？  
我们知道，每个用户在 100 次的并发操作中，key 的值是一样的，但是架不住人家要比较 key 的内存地址啊！

调用`String#intern()`便是解决了这个问题，那么这个函数到底有何魔力呢？来，点进去看看源码：

```java
/** 
 * Returns a canonical representation for the string object. 
 * <p> 
 * A pool of strings, initially empty, is maintained privately by the 
 * class <code>String</code>. 
 * <p> 
 * When the intern method is invoked, if the pool already contains a 
 * string equal to this <code>String</code> object as determined by 
 * the {@link ##equals(Object)} method, then the string from the pool is 
 * returned. Otherwise, this <code>String</code> object is added to the 
 * pool and a reference to this <code>String</code> object is returned. 
 * <p> 
 * It follows that for any two strings <code>s</code> and <code>t</code>, 
 * <code>s.intern() == t.intern()</code> is <code>true</code> 
 * if and only if <code>s.equals(t)</code> is <code>true</code>. 
 * <p> 
 * All literal strings and string-valued constant expressions are 
 * interned. String literals are defined in section 3.10.5 of the 
 * <cite>The Java™ Language Specification</cite>. 
 * 
 * @return  a string that has the same contents as this string, but is 
 *          guaranteed to be from a pool of unique strings. 
 */  
public native String intern();
```

根据javadoc描述，如果常量池中存在当前字符串, 就会直接返回当前字符串引用. 如果常量池中没有此字符串, 会将此字符串放入常量池中后, 再返回其引用，这样内存地址不就一样了嘛，哈哈。

## 警告 ##

上面的改造方法虽然简单，但是坑很大，需要注意：  
`String#intern()`在jdk6里问题不算大，会在perm里产生空间，如果perm空间够用的话，不会导致频繁Full GC,  
但是在jdk7里问题就大了，会在heap里产生空间，而且还是老年代，如果对象一多就会导致频繁Full GC！

因此，请慎用。

```bash
### 查看 gc
jstat -gcutil $(jps -l | grep keyword | awk -F  '{print $1}') 5000
```

## 再改造 ##

既然`String#intern()`如此危险，那么还有替代的解决方法吗？当然有：

```java
private static final Interner<String> mLockKeyPool = Interners.newWeakInterner();
static Integer get(String key) {
    synchronized (mLockKeyPool.intern(key)) {
        Integer count = COUNT_MAP.get(key);
        return null == count ? 0 : count;
    }
}

static void set(String key, Integer count) {
    synchronized (mLockKeyPool.intern(key)) {
        COUNT_MAP.put(key + "", count);
    }
}
```

很好，`Google Guava`帮我们解决了这一问题，使用`Interners.newWeakInterner()`即可。

```java
/**
 * Returns a new thread-safe interner which retains a weak reference to each instance it has
 * interned, and so does not prevent these instances from being garbage-collected. This most
 * likely does not perform as well as {@link ##newStrongInterner}, but is the best alternative
 * when the memory usage of that implementation is unacceptable. Note that unlike {@link
 * String#intern}, using this interner does not consume memory in the permanent generation.
*/
@GwtIncompatible("java.lang.ref.WeakReference")
public static <E> Interner<E> newWeakInterner() {
    return new WeakInterner<E>();
}
```

注意看 javadoc 的最后一句，【 **注意，与String\#intern()不同，使用这个interner不会消耗永生代中的内存** 】这下放心了，至于原理，下次再分析吧！