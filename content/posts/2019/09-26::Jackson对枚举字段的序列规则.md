+++
title = "Jackson对枚举字段的序列规则"
date = "2019-09-26 16:53:00"
url = "archives/654"
tags = ["Java","Jackson"]
categories = ["后端"]
+++

现有个枚举类型，内部定义了一些属性，使用 `Jaskson` 来进行序列化成 JSON 字符串

```kotlin
enum class Error(
        var code: Int,
        var reason: String,
        var description: String
) {
    UNKNOWN(-1, "unknown", "发生了未知错误，请联系管理员。");
}
```

看起来没什么问题，先直接序列化试试吧：

```kotlin
ObjectMapper().writerWithDefaultPrettyPrinter().writeValueAsString(Error.UNKNOWN)
```

然而序列化结果是这样的：

    "UNKNOWN"

哎，跟预期完全不符啊，查阅了资料，主要有几种解决方法，都比较麻烦，这里就记录一种最简单的：

```kotlin
// 添加该注解即可解决
@JsonFormat(shape = JsonFormat.Shape.OBJECT)
enum class Error(
        var code: Int,
        var reason: String,
        var description: String
) {
    UNKNOWN(-1, "unknown", "发生了未知错误，请联系管理员。");
}

ObjectMapper().writerWithDefaultPrettyPrinter().writeValueAsString(RestResponse.Error.UNKNOWN)

{
  "code" : -1,
  "reason" : "unknown",
  "description" : "发生了未知错误，请联系管理员。"
}
```

 *  参考链接：  
    [https://segmentfault.com/q/1010000006935643/a-1020000006936789][https_segmentfault.com_q_1010000006935643_a-1020000006936789]  
    [https://blog.csdn.net/z69183787/article/details/54292789][https_blog.csdn.net_z69183787_article_details_54292789]
 *  FastJson 参考链接：  
    [https://www.cnblogs.com/VergiLyn/p/6753706.html][https_www.cnblogs.com_VergiLyn_p_6753706.html]


[https_segmentfault.com_q_1010000006935643_a-1020000006936789]: https://segmentfault.com/q/1010000006935643/a-1020000006936789
[https_blog.csdn.net_z69183787_article_details_54292789]: https://blog.csdn.net/z69183787/article/details/54292789
[https_www.cnblogs.com_VergiLyn_p_6753706.html]: https://www.cnblogs.com/VergiLyn/p/6753706.html