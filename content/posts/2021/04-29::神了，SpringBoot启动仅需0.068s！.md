+++
title = "神了，SpringBoot启动仅需0.068s！"
date = "2021-04-29 09:26:55"
url = "archives/718"
tags = ["Spring","Java","Spring Native","GraalVM"]
categories = ["后端"]
+++

`GraalVM`是`Oracle`搞出来的一种高性能的虚拟机，可以显著的提高程序的性能和运行效率，非常适合微服务。

最近比较火的`Quarkus`框架默认支持，`SpringBoot`当前也开始跟上了步伐，在`Spring Initializr`网站可以看到，基于`GraalVM`的`Spring Native`已经进入了实验阶段，

![d5434fac-7e1e-4647-9733-5fa3bcaac10d.png][]

这意味着，不久的将来能提供一种全新的方式部署 Spring 应用，这些原生的 Spring 应用可以作为一个独立的可执行文件进行部署（不需要安装 JVM），并且还能瞬时的启动（一般会小于 100 毫秒）、瞬时的峰值性能以及更低的资源消耗，其代价是比 JVM 更长的构建时间和更少的运行时优化。

### 安装GraalVM（手动） ###

> 本文以macOS系统为例，详见官方文档：https://www.graalvm.org/docs/getting-started/\#install-graalvm

#### 下载graalvm releases包 ####

> 具体地址在https://github.com/graalvm/graalvm-ce-builds/releases

```bash
### 下载graalvm-jdk11包, 
wget https://github.com/graalvm/graalvm-ce-builds/releases/download/vm-21.1.0/graalvm-ce-java11-darwin-amd64-21.1.0.tar.gz
```

#### 解压缩并移动至JDK安装目录 ####

```bash
tar -xzf graalvm-ce-java11-darwin-amd64-21.1.0.tar.gz 
sudo mv graalvm-ce-java11-21.1.0 /Library/Java/JavaVirtualMachines

### macOS高版本执行
sudo xattr -r -d com.apple.quarantine /Library/Java/JavaVirtualMachines/graalvm-ce-java11-21.1.0/
```

#### 验证已经安装的JDK ####

```bash
/usr/libexec/java_home -V
```

![02e17bdc-6dff-4098-9823-c83e4278cb20.png][]

从上图可以看到，我本地电脑安装了3个不同版本的JDK，其中GraalVM已经成功安装。

#### 配置环境变量 ####

```bash
vi ~/.bash_profile
export JAVA_HOME=/Library/Java/JavaVirtualMachines/graalvm-ce-java11-21.1.0/Contents/Home
export PATH=$JAVA_HOME:$PATH
source ~/.bash_profile

java -version
openjdk version "16.0.1" 2021-04-20
OpenJDK Runtime Environment GraalVM CE 21.1.0 (build 16.0.1+9-jvmci-21.1-b05)
OpenJDK 64-Bit Server VM GraalVM CE 21.1.0 (build 16.0.1+9-jvmci-21.1-b05, mixed mode, sharing)
```

### 安装GraalVM（使用sdkman，推荐） ###

> 相比手动安装，使用sdkman（https://sdkman.io/）来得更方便一点

```bash
### 以macOS为例，其他系统环境请参考sdkman官方文档
### 安装sdkman
curl -s "https://get.sdkman.io" | bash
source "$HOME/.sdkman/bin/sdkman-init.sh"

### 查询可用的java安装包
sdk list java

### 安装graalvm最新版
sdk install java 21.1.0.r16-grl

### 设置graalvm为默认jdk
sdk default java 21.1.0.r16-grl

### 验证jdk版本
java -version
openjdk version "16.0.1" 2021-04-20
OpenJDK Runtime Environment GraalVM CE 21.1.0 (build 16.0.1+9-jvmci-21.1-b05)
OpenJDK 64-Bit Server VM GraalVM CE 21.1.0 (build 16.0.1+9-jvmci-21.1-b05, mixed mode, sharing)
```

### 安装 native-image ###

除此之外，还需要安装`native-image`（GraalVM的子项目）来编译Spring Boot程序。

遗憾的是，GraalVM并未内置该工具，因此我们还需要多执行一行代码：

```bash
cd $JAVA_HOME/bin
### 使用GraalVM更新器来安装native-image
./gu install native-image
```

![c9f5d6f1-cf18-4e0f-89f9-571ce4ede4bf.png][]

### **准备Spring Boot应用** ###

为了便于比对测试，咱们分别准备springboot应用，均实现一个`/sayHello`接口，以待后续观察

 *  normal-springboot-demo
 *  graalvm-springboot-demo

```java
package com.example.normalspringbootdemo;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
@SpringBootApplication
public class NormalSpringbootDemoApplication {

    public static void main(String[] args) {
        SpringApplication.run(NormalSpringbootDemoApplication.class, args);
    }

    @GetMapping("/sayHello")
    public ResponseEntity<String> sayHello() {
        return ResponseEntity.ok("Hello World!");
    }
}
```

![4ca51d80-9bbf-4b88-bea8-bbcff71629d3.png][]

#### 运行normal-springboot-demo，观察内存占用及启动速度 ####

```bash
cd normal-springboot-demo 
./mvnw package
java -jar target/normal-springboot-demo-0.0.1-SNAPSHOT.jar
```

![07a83662-cbfc-4d17-adf8-5ab359fa15ff.png][]

执行以下命令来查看JAVA程序的内存占用情况：

```bash
ps aux | grep normal-springboot-demo-0.0.1-SNAPSHOT.jar | grep -v grep | awk '{print $11 "t" $6/1024"MB" }'
```

![5e468cb8-6228-4807-8d1e-c29db242fc20.png][]

好家伙，我直接好家伙！

一个近乎空白的SpringBoot应用启动时间达到了`2.1s`，而内存占用更是达到了惊人的`254.516MB`，不愧是内存大户。

#### 使用 GraalVM 打包并运行graalvm-springboot-demo ####

先别着急直接打包运行，在打包先，还得做以下几件事情，让springboot应用支持

 *  `native-image` 本地映像打包插件 （前文已经安装）

```bash
### 验证
native-image --version
```

 *  修改GraalvmSpringbootDemoApplication启动类

> Graal VM不支持CGLIB，只能使用JDK动态代理，所以应当把Spring对普通类的Bean增强给关闭掉

```java
@RestController
// 在此将proxyBeanMethods方法代理关闭
@SpringBootApplication(proxyBeanMethods = false)
public class GraalvmSpringbootDemoApplication {

    public static void main(String[] args) {
        SpringApplication.run(GraalvmSpringbootDemoApplication.class, args);
    }

    @GetMapping("/sayHello")
    public ResponseEntity<String> sayHello() {
        return ResponseEntity.ok("Hello World!");
    }
}
```

 *  配置Maven仓库为Spring官方

```xml
<repositories>
    <repository>
        <id>spring-releases</id>
        <name>Spring Releases</name>
        <url>https://repo.spring.io/release</url>
        <snapshots>
            <enabled>false</enabled>
        </snapshots>
    </repository>
</repositories>
<pluginRepositories>
    <pluginRepository>
        <id>spring-releases</id>
        <name>Spring Releases</name>
        <url>https://repo.spring.io/release</url>
        <snapshots>
            <enabled>false</enabled>
        </snapshots>
    </pluginRepository>
</pluginRepositories>
```

 *  升级 `springboot` 版本为2.4.5

```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>2.4.5</version>
    <relativePath/> <!-- lookup parent from repository -->
</parent>
```

 *  配置`native-image-maven-plugin`

```xml
<profiles>
    <profile>
        <id>native</id>
        <build>
            <plugins>
                <plugin>
                    <groupId>org.graalvm.nativeimage</groupId>
                    <artifactId>native-image-maven-plugin</artifactId>
                    <version>21.0.0</version>
                    <configuration>
                        <mainClass>com.example.graalvmspringbootdemo.GraalvmSpringbootDemoApplication</mainClass>
                    </configuration>
                    <executions>
                        <execution>
                            <goals>
                                <goal>native-image</goal>
                            </goals>
                            <phase>package</phase>
                        </execution>
                    </executions>
                </plugin>
            </plugins>
        </build>
    </profile>
</profiles>


<build>
    <plugins>
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
            <configuration>
                <classifier>exec</classifier>
                <image>
                    <builder>paketobuildpacks/builder:tiny</builder>
                    <env>
                        <BP_NATIVE_IMAGE>true</BP_NATIVE_IMAGE>
                    </env>
                </image>
            </configuration>
        </plugin>
        <plugin>
            <groupId>org.springframework.experimental</groupId>
            <artifactId>spring-aot-maven-plugin</artifactId>
            <version>0.9.2</version>
            <executions>
                <execution>
                    <id>test-generate</id>
                    <goals>
                        <goal>test-generate</goal>
                    </goals>
                </execution>
                <execution>
                    <id>generate</id>
                    <goals>
                        <goal>generate</goal>
                    </goals>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

 *  使用 `native-image`构建可执行文件

```bash
cd graalvm-springboot-demo
./mvnw -Pnative package
```

![1333f747-a507-46f2-9df3-32fc94277ade.png][]

可以看到，构建非常的慢，共花费约5分钟时间。

 *  执行可执行文件，观察内存占用以及启动时间

```bash
./target/com.example.graalvmspringbootdemo.graalvmspringbootdemoapplication
```

![e5cad9b9-9be8-4dc3-987e-16fb4aed3b6f.png][]

```bash
ps aux | grep com.example.graalvmspringbootdemo.graalvmspringbootdemoapplication | grep -v grep | awk '{print $11 "t" $6/1024"MB" }'
```

![c57394be-1f20-4b7c-b9b8-2c5ea7bad8fc.png][]

内存占用和启动时间有了明显的提升

|                 | 启动时间   | 内存占用      |
|-----------------|--------|-----------|
| 不使用             | 2.1s   | 254.516MB |
| 使用Spring Native | 0.068s | 53.226MB  |

最后，由于`Spring Native`目前处于实验阶段，还不支持 `CGLIB`的动态代理，谨慎入坑吧，期待早日迎来正式版本。


[d5434fac-7e1e-4647-9733-5fa3bcaac10d.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210429/d5434fac-7e1e-4647-9733-5fa3bcaac10d.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[02e17bdc-6dff-4098-9823-c83e4278cb20.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210429/02e17bdc-6dff-4098-9823-c83e4278cb20.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[c9f5d6f1-cf18-4e0f-89f9-571ce4ede4bf.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210429/c9f5d6f1-cf18-4e0f-89f9-571ce4ede4bf.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[4ca51d80-9bbf-4b88-bea8-bbcff71629d3.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210429/4ca51d80-9bbf-4b88-bea8-bbcff71629d3.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[07a83662-cbfc-4d17-adf8-5ab359fa15ff.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210429/07a83662-cbfc-4d17-adf8-5ab359fa15ff.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[5e468cb8-6228-4807-8d1e-c29db242fc20.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210429/5e468cb8-6228-4807-8d1e-c29db242fc20.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[1333f747-a507-46f2-9df3-32fc94277ade.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210429/1333f747-a507-46f2-9df3-32fc94277ade.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[e5cad9b9-9be8-4dc3-987e-16fb4aed3b6f.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210429/e5cad9b9-9be8-4dc3-987e-16fb4aed3b6f.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[c57394be-1f20-4b7c-b9b8-2c5ea7bad8fc.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210429/c57394be-1f20-4b7c-b9b8-2c5ea7bad8fc.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg