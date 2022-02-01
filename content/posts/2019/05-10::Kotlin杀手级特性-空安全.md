+++
title = "Kotlin杀手级特性-空安全"
date = "2019-05-10 16:19:00"
url = "archives/566"
tags = ["Kotlin"]
categories = ["后端"]
+++

Kotlin相对于Java来说，有一个显著的特点，就是它致力于消除空引用所带来的危险，在Java中，为了避免`NullPointerException`的出现，我们需要不厌其烦的使用`if (value != null) {} `来处理这种问题（虽然在JDK8之后有了更好的方式）

在Kotlin中很好解决了这个问题，下面来看看它是如何做到的。

### 数据类型 ###

数据类型分为`非空类型`和`可空类型`

 *  非空类型：Char, Boolean, Byte, Short, Int，Long, Float, Double,String
 *  可空类型：Char?, Boolean?, Byte?, Short?, Int?，Long?, Float?, Double?, String?

![2019-08-19-073124][]  
上图这段代码中，User对象中的name，email字段都定义成了非空类型，在下面初始化值时，就不能赋值为null  
简直太粗暴了，但是至少现在看来，我们不需要在使用user.email时去判断是否为空了。

### 空安全运算符 ###

#### 安全调用`(?.)` ####

继续改进上段代码，将`email`的类型定义为`String?`，这样就能赋值为null了，但是使用呢？又回到了if判空的尴尬局面吗？显然不是

```kotlin
open class User(val name: String, val email: String?)

fun testNullUserEmail() {
    var user = User("张三", null)

    var emailLength = user.email?.length
    println("length = $emailLength") // length = null
}

fun main(args: Array<String>) {
    testNullUserEmail()
}
```

改进后的代码使用了`?.`安全调用，因为email为空，则不再调用`length`属性，而是直接返回了null，现在可以肯定的讲，这段代码一辈子都不会抛出`NullPointerException`了。

#### Elvis运算符`(?:)` ####

还是那段代码，如果user.email为null，我想干一些其他的事情咋办？

```kotlin
import java.lang.IllegalArgumentException

/** * @author wuwenze * @date 2019-05-10 */

open class User(val name: String, val email: String?)

fun testNullUserEmail() {
    var user = User("张三", null)

    // var emailLength = user.email?.length
    var emailLength = user.email?.length ?: 0 // 如果为空, 返回0
    println("length = $emailLength")

    // 或者抛出一个预定义的异常
    var len = user.email?.length ?: throw IllegalArgumentException("user.email is null")

    // 执行其他的逻辑操作？
    var len2 = user.email?.length ?: println("user.email is null")
    println("len2 = $len2") // len2 = kotlin.Unit

    // 中断函数的执行？
    var len3 = user.email?.length ?: return
}

fun main(args: Array<String>) {
    testNullUserEmail()
}
```

#### 非空断言运算符`(!!)` ####

再来看一段代码：定义类型为可空类型(String?)的name字符串  
![2019-08-19-073125][]  
下面在调用length时，居然编译不通过了，说可能有空指针的风险（真是安逸啊）

现在开发者明确的知道，name不可能为空，有两种办法，一是修改数据类型为非空类型，二是使用`!!`运算符把任何值转换成非空类型

```kotlin
fun main(args: Array<String>) {
    var name :String? = "hello"

    var len = name!!.length
    println("$name length is $len") // hello length is 5
}
```

#### 安全类型转换`(as?)` ####

我们知道类型转换可能产生`ClassCastException`异常，例如：

```kotlin
var a: Long = 1
val aInt: Int? = a as Int  // java.lang.ClassCastException
```

使用`as?`优化一下代码即可解决

```kotlin
var a: Long = 1
val aInt: Int? = a as? Int // 如果类型转换不成功，返回null
```

#### let函数`(.let)` ####

再来看一段代码， 这段代码中，定义了一个getNullUser函数，这段代码可能返回空，那么在之前java的做法可能类似于这样，尽管我现在用Kotlin如此简洁的语言来写，还是感觉诺里啰嗦，更别说Java了

```kotlin
open class User(val name: String?, val email: String?)

fun getNullUser(): User? = null

fun main(args: Array<String>) {
    val user = getNullUser()

    if (user != null && user.name != null) {
        println("user.name = ${user.name}, user.name.len = ${user.name.length}")
    }
    if (user != null && user.email != null) {
        println("user.email = ${user.email}, user.email.len = ${user.email.length}")
    }
}
```

使用let函数改造一下吧：

```kotlin
open class User(val name: String?, val email: String?)

fun getNullUser(): User? = null

fun main(args: Array<String>) {
    getNullUser().let {
        println("user.name = ${it?.name}, user.name.len = ${it?.name?.length}")
        println("user.email = ${it?.email}, user.email.len = ${it?.email?.length}")
    }
}
```

这段代码不会有任何的输出，因为getNullUser()函数始终返回的是一个NULL，let代码块里面的代码不会被执行。


[2019-08-19-073124]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20190510/51589315-949d-44aa-aa6f-c9751d087935.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[2019-08-19-073125]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20190510/8d24d381-bf91-480e-9b31-b9646f25ca3b.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg