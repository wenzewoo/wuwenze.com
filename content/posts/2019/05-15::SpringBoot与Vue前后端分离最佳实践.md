+++
title = "SpringBoot与Vue前后端分离最佳实践"
date = "2019-05-15 16:21:00"
url = "archives/572"
tags = ["Spring","Vue"]
categories = ["后端"]
+++

前后端分离的开发模式大家都很清楚了，甚是麻烦：

 *  前端启动webpack-dev-server
 *  后端启动接口服务
 *  开启代理服务器，前端通过代理服务器请求后端接口（解决跨域问题）

但是这些东西对于后端来说，太麻烦了，直接把前端打包好的dist文件丢到后端静态服务器里面就好了。  
至于前端的webpack-dev-server热部署特性，改完前端代码立即在浏览器生效，对于后端来说有什么用呢？

 *  配置前端项目打包Task
 *  配置后端项目启动Task（执行前端编译、拷贝编译后的静态文件、启动后端）
 *  一键启动

### 本地环境 ###

 *  \*\*IntelliJ IDEA：\*\*2019.1 (Ultimate Edition)
 *  **Gradle**：5.4.1

### 创建项目 ###

使用Gradle来构建项目，他的Task构建任务非常灵活，远胜Maven。

#### 创建项目 ####

![image.png][]

注意，这里不要勾选任何库和框架（只勾选了Kotlin DSL，因为Gradle默认使用Groovy来描述配置文件，不太熟悉这个脚本语言，还是使用Kotlin吧）

![image.png][image.png 1]

项目创建成功，这时是一个空的项目，在里面加入后端和前端模块

#### 创建前端模块 ####

![image.png][image.png 2]

同样的，不要勾选任何框架，只选择使用Kotlin DSL就行了。  
![image.png][image.png 3]

![image.png][image.png 4]

修改`settings.gradle.kts`，将新建的前端项目作为模块引入进来：

```bash
rootProject.name = "springboot-vue-gradle"
include("webui")
```

打开终端，使用vue-cli初始化前端项目：  
![image.png][image.png 5]

```bash
~ cd Development/other-projects/springboot-vue-gradle 
  springboot-vue-gradle vue init webpack webui

? Target directory exists. Continue? Yes
? Project name webui
? Project description A Vue.js project
? Author 吴汶泽 <wuwz@live.com>
? Vue build standalone
? Install vue-router? Yes
? Use ESLint to lint your code? Yes
? Pick an ESLint preset Standard
? Set up unit tests No
? Setup e2e tests with Nightwatch? No
? Should we run `npm install` for you after the project has been created? (recom
mended) yarn
```

注意，这里初始化前端项目时，文件夹名称要覆盖掉之前创建的前端模块（webui）构建好的文件目录结构如下：  
![image.png][image.png 6]

前端项目暂告一段落，现在开始创建后端项目

#### 创建后端模块 ####

![image.png][image.png 7]

这里使用`Spring Initializer`，免去一些配置SpringBoot的麻烦事儿  
![image.png][image.png 8]

配置项目，选用Gradle构建，并使用Kotlin语言  
![image.png][image.png 9]

选择项目依赖，由于是演示，这里就只选了WEB，数据库这些都不弄了  
![image.png][image.png 10]

修改`settings.gradle.kts`，将新建的后端项目作为模块引入进来：

```bash
rootProject.name = "springboot-vue-gradle"
include("webui", "server")
```

构建的后端项目现在还没有生成文件夹和启动类，需要我们自己手动创建一下：  
![image.png][image.png 11]

创建后的效果图如下：  
![image.png][image.png 12]

先编写一个简单的后端接口：

```java
@RestController
@SpringBootApplication
class Application {
    companion object {
        @JvmStatic fun main(args: Array<String>) {
            runApplication<Application>(*args)
        }
    }

    @GetMapping("/sayHello")
    fun sayHello(): Map<String, Any?> {
        return mapOf("message" to "Hello, Springboot & Vue")
    }
}
```

![image.png][image.png 13]

#### 编写前端构建任务 ####
> springboot-vue-gradle/webui/build.gradle.kts

```kotlin
import com.moowork.gradle.node.yarn.YarnTask

group = "com.wuwenze"
version = "1.0-SNAPSHOT"

plugins {
  id("com.moowork.node") version "1.3.1"
}

tasks {
  // start the webui app （以webpack-dev-server方式）
  register<YarnTask>("startWebUI") {
    group = "application"
    setYarnCommand("run", "start")
  }

  // build the webui app
  register<YarnTask>("buildWebUI") {
    group = "application"
    setYarnCommand("run", "build")
  }
}
```

使用Gradle编写两个构建命令，一个是启动前端，一个是编译前端，其本质是调用yarn命令  
![image.png][image.png 14]

#### 编写项目构建任务 ####
> springboot-vue-gradle/build.gradle.kts

```kotlin
group = "com.wuwenze"
version = "1.0-SNAPSHOT"

tasks {
    register<Copy>("copyWebDist") {
        group = "application"

        dependsOn("webui:buildWebUI")
        from("webui/dist")
        into("server/src/main/resources/static")
    }

    // 启动开发服务，先编译前端项目，再启动SpringBoot
    register("startDevServer") {
        group = "application"

        dependsOn(":copyWebDist", "server:bootRun")
    }
}
```

![image.png][image.png 15]

前后端算是无缝打通了

对于后端人员来说，直接启动startDevServer就行了  
不需要单独启动前端，再启动后端，最后在来个代理服务器（太麻烦了）

### 执行测试 ###

之前在后端提供了一个`/sayHello`接口，在前端调用这个接口试试。

```bash
cd webui/
yarn add axios --save
```

改造一下前端的`webui/src/components/HelloWorld.vue`

```html
<script>
  import axios from "axios";
  export default {
    name: 'HelloWorld',
    data() {
      return {
        msg: 'Welcome to Your Vue.js App'
      }
    },
    created() {
      axios.get("/sayHello").then((res) => this.msg = res.data.message)
    }
  }
</script>
```

启动项目：  
![image.png][image.png 16]

浏览器访问：`localhost:10054`  
![image.png][image.png 17]


[image.png]: https://wuwenze-1253645550.cos.ap-chengdu.myqcloud.com/2019-08-19-073156.png
[image.png 1]: https://wuwenze-1253645550.cos.ap-chengdu.myqcloud.com/2019-08-19-073159.png
[image.png 2]: https://wuwenze-1253645550.cos.ap-chengdu.myqcloud.com/2019-08-19-073200.png
[image.png 3]: https://wuwenze-1253645550.cos.ap-chengdu.myqcloud.com/2019-08-19-073201.png
[image.png 4]: https://wuwenze-1253645550.cos.ap-chengdu.myqcloud.com/2019-08-19-073202.png
[image.png 5]: https://wuwenze-1253645550.cos.ap-chengdu.myqcloud.com/2019-08-19-073203.png
[image.png 6]: https://wuwenze-1253645550.cos.ap-chengdu.myqcloud.com/2019-08-19-73204.png
[image.png 7]: https://wuwenze-1253645550.cos.ap-chengdu.myqcloud.com/2019-08-19-073205.png
[image.png 8]: https://wuwenze-1253645550.cos.ap-chengdu.myqcloud.com/2019-08-19-073206.png
[image.png 9]: https://wuwenze-1253645550.cos.ap-chengdu.myqcloud.com/2019-08-19-073208.png
[image.png 10]: https://wuwenze-1253645550.cos.ap-chengdu.myqcloud.com/2019-08-19-073209.png
[image.png 11]: https://wuwenze-1253645550.cos.ap-chengdu.myqcloud.com/2019-08-19-073210.png
[image.png 12]: https://wuwenze-1253645550.cos.ap-chengdu.myqcloud.com/2019-08-19-073211.png
[image.png 13]: https://wuwenze-1253645550.cos.ap-chengdu.myqcloud.com/2019-08-19-73213.png
[image.png 14]: https://wuwenze-1253645550.cos.ap-chengdu.myqcloud.com/2019-08-19-073214.png
[image.png 15]: https://wuwenze-1253645550.cos.ap-chengdu.myqcloud.com/2019-08-19-073216.png
[image.png 16]: https://wuwenze-1253645550.cos.ap-chengdu.myqcloud.com/2019-08-19-073218.png
[image.png 17]: https://wuwenze-1253645550.cos.ap-chengdu.myqcloud.com/2019-08-19-73219.png