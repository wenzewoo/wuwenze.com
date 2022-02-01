+++
title = "使用Jsoup处理富文本XSS攻击过滤"
date = "2019-08-28 16:49:00"
url = "archives/647"
tags = ["Java","XSS"]
categories = ["后端"]
+++

由于富文本的特殊性，不能一刀切将所有的 HTML 标签转义，但为了防止 XSS 攻击, 又必须过滤掉其中的 JS 代码, 在 Java 中使用 Jsoup 正好可以满足此要求。

Jsoup 采用白名单过滤的方式，只允许配置的白名单标签进行提交，故需要好好整理一下，到底哪些标签是可以使用的。

### 实现 ###

```xml
<dependency>
    <groupId>org.jsoup</groupId>
    <artifactId>jsoup</artifactId>
    <version>1.9.2</version>
</dependency>
```

```kotlin
import org.jsoup.Jsoup
import org.jsoup.nodes.Document
import org.jsoup.safety.Whitelist

fun main() {
    // 使用预置的白名单
    //val whitelist: Whitelist = Whitelist.relaxed() // 常见的 HTML 标签
    // .addAttributes(":all", "id", "class", "style") // 所有标签都允许的属性

    // 自定义白名单
    val whitelist: Whitelist = Whitelist()
            .addTags("a", "b", "blockquote", "br", "caption", "cite", "code", "col", "colgroup", "dd", "div", "dl", "dt",
                    "em", "h1", "h2", "h3", "h4", "h5", "h6", "i", "img", "li", "ol", "p", "pre", "q", "small", "span", "strike",
                    "strong", "sub", "sup", "table", "tbody", "td", "tfoot", "th", "thead", "tr", "u", "ul")
            .addAttributes("a", "href", "title")
            .addAttributes("blockquote", "cite")
            .addAttributes("col", "span", "width")
            .addAttributes("colgroup", "span", "width")
            .addAttributes("img", "align", "alt", "height", "src", "title", "width")
            .addAttributes("ol", "start", "type")
            .addAttributes("q", "cite")
            .addAttributes("table", "summary", "width")
            .addAttributes("td", "abbr", "axis", "colspan", "rowspan", "width")
            .addAttributes("th", "abbr", "axis", "colspan", "rowspan", "scope", "width")
            .addAttributes("ul", "type")
            .addAttributes(":all", "id", "class", "style")
            .addProtocols("a", "href", "ftp", "http", "https", "mailto")
            .addProtocols("blockquote", "cite", "http", "https")
            .addProtocols("cite", "cite", "http", "https")
            .addProtocols("img", "src", "http", "https")
            .addProtocols("q", "cite", "http", "https");


    val bodyHtml = """
        <div class="div-test"><img src="123" onerror="javascript:alert(123);"></div>
    """.trimIndent()

    println(Jsoup.clean(bodyHtml, whitelist))

    println("--------------")

    // 不格式化输出
    println(Jsoup.clean(bodyHtml, "", whitelist, Document.OutputSettings().prettyPrint(false)))
}
```

```bash
<div class="div-test">
 <img>
</div>
--------------
<div class="div-test"><img></div>

Process finished with exit code 0
```

### 预置的白名单 ###

| 预置                | 标签                                                                                                                                                                            | 说明                                |
|-------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------|
| none()            | 无                                                                                                                                                                             | 过滤所有 HTML 代码，只保留纯文本               |
| simpleText()      | b,em,i,strong,u                                                                                                                                                               | 只允许简单的文本标签                        |
| basic()           | a,b,blockquote,br,cite,code,dd,dl,dt,em,i,li,ol,p,pre,q,small,span,strike,strong,sub,sup,u,ul                                                                                 | 基本使用的标签                           |
| basicWithImages() | 在基本使用的标签基础上，又增加了图片标签以及src,align,alt,height,width,title 属性                                                                                                                     | 基本+图片                             |
| relaxed()         | a,b,blockquote,br,caption,cite,code,col,colgroup,dd,div,dl,dt,em,h1,h2,h3,h4,h5,h6,i,img,li,ol,p,pre,q,small,span,strike,strong,sub,sup,table,tbody,td,tfoot,th,thead,tr,u,ul | 在 basicWithImages 的基础上又增加了一部分部分标签 |
