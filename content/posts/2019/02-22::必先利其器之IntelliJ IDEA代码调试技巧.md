+++
title = "必先利其器之IntelliJ IDEA代码调试技巧"
date = "2019-02-22 08:19:00"
url = "archives/493"
tags = ["Intellij IDEA"]
categories = ["开发工具"]
+++

### Debug 设置 ###

![2019-08-19-73756][]

一般来说，保持默认即可，如果在Windows环境下，建议将图中标记的地方（Debug连接的方式）选择为` Shared memory`，该选项是Windows下才有的特性，相比Socket的方式来说，要快不少。

### 常用快捷键 ###

| 快捷键               | 介绍                                                                    |
|-------------------|-----------------------------------------------------------------------|
| F7                | 在 Debug 模式下，进入下一步，如果当前行断点是一个方法，则进入当前方法体内，如果该方法体还有方法，则不会进入该内嵌的方法中 必备   |
| F8                | 在 Debug 模式下，进入下一步，如果当前行断点是一个方法，则不进入当前方法体内 必备                          |
| F9                | 在 Debug 模式下，恢复程序运行，但是如果该断点下面代码还有断点则停在下一个断点上 必备                        |
| Alt + F8          | 在 Debug 的状态下，选中对象，弹出可输入计算表达式调试框，查看该输入内容的调试结果 必备                       |
| Ctrl + F8         | 在 Debug 模式下，设置光标当前行为断点，如果当前已经是断点则去掉断点                                 |
| Shift + F7        | 在 Debug 模式下，智能步入。断点所在行上有多个方法调用，会弹出进入哪个方法                              |
| Shift + F8        | 在 Debug 模式下，跳出，表现出来的效果跟 F9 一样                                         |
| Ctrl + Shift + F8 | 在 Debug 模式下，指定断点进入条件                                                  |
| Alt + Shift + F7  | 在 Debug 模式下，进入下一步，如果当前行断点是一个方法，则进入当前方法体内，如果方法体还有方法，则会进入该内嵌的方法中，依此循环进入 |
| Drop Frame        | 这个不是一个快捷键，而是一个 Debug 面板上的按钮。该按钮可以用来退回到当前停住的断点的上一层方法上，可以让过掉的断点重新来过     |


### 工具栏详解 ###

![fac33409-0f4f-4962-b508-8df0412d1137.png][]

上图通过数字标记的工具栏是在调试过程中需要经常使用的，他们的作用分别如下：  
\*\*1）\*\*进入下一步，如果是方法，那就直接跳过（F8）  
\*\*2）\*\*进入下一步，如果是方法，就进入方法内部，但是不会进入jdk封装的方法（F7）  
\*\*3）\*\*强制进入下一步，不管是什么方法，即使是jdk封装的方法，也会进入（Alt+Shift+F7）  
\*\*4）\*\*跳转到下一个断点，如果没有，那就一直运行到最后（Shift + F8）  
\*\*5）\*\*运行程序到光标所在的行（Alt + F9）

### 调试技巧 ###

#### Evaluate（Alt + F8） ####

该工具允许在程序调试的过程中，计算指定表达式的值，这在实际调试过程中，是相当重要的功能。

![7f5ec565-b14f-486e-a25f-919bcf385b41.gif][]

上图中还展示一个Evaluate的一个重要特性，除了一些简单的表达式外，还能执行一些指定代码片段然后求出最后一行代码的值。

#### Watches ####

在调试过程中，动态修改某个变量的值

![da4bff08-a59c-44b2-96a3-039006af871b.gif][]

#### 配置断点步入条件 ####

![cac2281d-d7aa-474f-9c98-4a0c2401f35f.gif][]

如图所示在循环中设置了一个断点，并在断点上配置表达式，当循环变量`number == 300`时，断点才会被挂起。

#### 异常重现 ####

![ff52a137-ea36-4531-b406-07ff0a8caa57.gif][]

IntelliJ IDEA 的部分调试技巧暂时介绍到这里，这些都只是冰山一角，如果你决定抛弃Eclipse转而使用IDEA，我相信这些特性会让你爱不释手且欲罢不能。


[2019-08-19-73756]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20190222/4ae276f7-a0ed-441e-ab46-8fa08a901afe.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[fac33409-0f4f-4962-b508-8df0412d1137.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20190222/fac33409-0f4f-4962-b508-8df0412d1137.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[7f5ec565-b14f-486e-a25f-919bcf385b41.gif]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20190222/7f5ec565-b14f-486e-a25f-919bcf385b41.gif?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[da4bff08-a59c-44b2-96a3-039006af871b.gif]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20190222/da4bff08-a59c-44b2-96a3-039006af871b.gif?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[cac2281d-d7aa-474f-9c98-4a0c2401f35f.gif]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20190222/cac2281d-d7aa-474f-9c98-4a0c2401f35f.gif?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[ff52a137-ea36-4531-b406-07ff0a8caa57.gif]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20190222/ff52a137-ea36-4531-b406-07ff0a8caa57.gif?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg