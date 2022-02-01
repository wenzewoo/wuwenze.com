+++
title = "使用Docker容器部署SpringBoot项目"
date = "2018-12-26 05:21:00"
url = "archives/62"
tags = ["Docker","Spring","Devops"]
categories = ["运维"]
+++

### Docker简介 ###

Docker是基于Go语言实现的云开源项目，诞生于2013年初，最初发起者是dotClouw公司。Docker 自开源后受到广泛的关注和讨论，目前已有多个相关项目，逐断形成了围Docker的生态体系。dotCloud 公司后来也改名为Docker Ine。

Docker是一个开源的容器引擎，它有助于更快地交付应用。 Docker可将应用程序和基础设施层隔离，并且能将基础设施当作程序一样进行管理。使用 Docker可更快地打包、测试以及部署应用程序，并可以缩短从编写到部署运行代码的周期。

> 官网地址：([https://docs.docker.com/][https_docs.docker.com]) 中文：([http://www.docker.org.cn/][http_www.docker.org.cn])

### Docker架构 ###

![98beb935-9147-42b4-96c6-d6d4c5e10a56.png][]

**`Docker daemon`**：运行在宿主机(DOCKER-HOST)的后台进程，可通过 Docker 客户端与之通信。

**`Images`**：一个只读的镜像模板，可以自己创建一个镜像也可以从网站上下载镜像供自己使用，镜像包含了一个RFS，一个镜像可以创建很多容器。

**`Container`**：由Docker Client通过镜像创建的实例，用户在容器中运行应用，一旦创建后就可以看做是一个简单的RFS，每个应用运行在隔离的容器中，享用独自的权限，用户，网络。确保安全与互相干扰两者在创建后，都是一堆layer的统一视角，唯一的却别是镜像最上面那一层是只读的，不可以修改，但是容器最上面一层是rw的，提供给用户操作。

**`Repository`**：镜像仓库。

### Docker VS 传统虚拟机 ###

Docker作为一种轻量级的虚拟化方式，Docker在运行应用上跟传统的虚拟机方式相比具有显著优势： 

 *  Docker容器很快，启动和停止可以在秒级实现，这相比传统的虚拟机方式要快得多。
 *  Docker容器对系统资源需求很少，一台主机上可以同时运行数千个Docker容器。
 *  Docker通过类似Git的操作来方便用户获取、分发和更新应用镜像，指令简明，学习成本较低。
 *  Docker通过Dockerfile配置文件来支持灵活的自动化创建和部署机制，提高工作效率。

| 特征   | Docker    | 虚拟机    |
|------|-----------|--------|
| 启动速度 | 秒级        | 分钟级    |
| 硬盘占用 | 一般为MB级    | 一般为GB级 |
| 性能   | 接近原生      | 低于原生   |
| 支持量  | 单机支持上千个容器 | 一般几十个  |
| 隔离性  | 完全隔离      | 完全隔离   |


Docker vs 传统虚拟机

### Docker环境安装 ###

> Docker 是一个开源的商业产品，有两个版本：社区版（Community Edition，缩写为 CE）和企业版（Enterprise Edition，缩写为 EE）

Docker要求`Linux内核版本在3.10以上`，本文中使用的是`CentOS 7.x`进行安装。

#### 检查内核版本 ####

```bash
uname -r
3.10.0-957.1.3.el7.x86_64
```

#### 更新yum包 ####

确保使用`ROOT权限`进行操作，否则可能导致不可预知问题。

```bash
yum -y update
```

#### 卸载旧版本 ####

若未安装过，忽略该步骤

```bash
yum remove docker docker-common docker-selinux docker-engine
```

#### 安装依赖包 ####

yum-util 提供yum-config-manager功能，另外两个是devicemapper驱动依赖的

```bash
yum install -y yum-utils device-mapper-persistent-data lvm2
```

#### 设置yum源 ####

```bash
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
```

#### 查看yum源中可供安装的版本 ####

```bash
yum list docker-ce --showduplicates | sort -r
```

#### 安装Docker ####

不指定版本，默认安装最新的stable版

```bash
sudo yum install -y docker-ce
```

#### 启动 & 加入开机启动 ####

```bash
systemctl start docker
systemctl enable docker
```

#### 验证Version ####

```bash
docker version
```

![cce73947-e8af-4e67-baf3-680749826e39.png][]

出现如上图提示，则表示Docker安装成功。

### Docker镜像相关命令 ###

#### 搜索镜像 ####

```bash
docker search java
```

搜索存放在Docker Hub中的镜像。

Docker Hub 官网： [https://hub.docker.com/search?q=java&type=image][https_hub.docker.com_search_q_java_type_image]

#### 下载镜像 ####

```bash
docker pull java:8
```

执行该命令后，Docker会从Docker Hub中的java仓库下载指定版本的Java镜像。

#### 查看已安装镜像 ####

```bash
docker images
```

![2019-08-19-73749][]

#### 删除镜像 ####

使用IMAGE ID进行删除，参考上一个命令

```bash
docker rmi d23bdf5b1b1b
```

#### 附：阿里云镜像加速服务 ####

[https://cr.console.aliyun.com/cn-hangzhou/mirrors][https_cr.console.aliyun.com_cn-hangzhou_mirrors]

### Docker容器相关命令 ###

#### 启动容器 ####

```bash
docker run -d -p 81:80 nginx
### -d 后台运行
### -p 宿主机端口:容器端口
```

### 开放容器端口到宿主机端口 ###

需要注意的是，使用docker run命令启动容器时，若本地没有相关镜像，Docker会从Docker Hub中自动下载相关镜像并启动

![2019-08-19-073749][]

访问 http://Docker宿主机IP:81/，将会看到Nginx的主界面：

![7fa2ab0b-9b83-4717-a0a2-0a7df5c811cc.png][]

#### 查看运行中的容器 ####

```bash
docker ps
```

#### 查看指定容器详细信息 ####

```bash
docker inspect b3e76480a94c
```

![148c41c3-0aa7-4629-a029-b54ea6637848.png][]

### 构建自定义Docker镜像 ###

为方便演示，这里创建一个简单的SpringBoot项目（docker-springboot-example）

```java
@RestController
@SpringBootApplication
public class DockerSpringbootExampleApplication {

    public static void main(String[] args) {
        SpringApplication.run(DockerSpringbootExampleApplication.class, args);
    }

    @GetMapping("/sayHello")
    public @ResponseBody ResponseEntity<?> sayHello() {
        return ResponseEntity.ok("Hello Docker & SpringBoot");
    }
}
```

#### 打包插件配置 ####

```xml
<plugin>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-maven-plugin</artifactId>
    <configuration>
     <mainClass>com.wuwenze.DockerSpringbootExampleApplication</mainClass>
    </configuration>
    <executions>
        <execution>
            <goals>
                <goal>repackage</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

#### 创建Dockerfile ####

文件暂时存放在Springboot项目的 src/main/resources 中，后面会用到。

```bash
##### 指定基础镜像
FROM java:8

##### 复制文件到容器
ADD docker-springboot-example-0.0.1-SNAPSHOT.jar /docker-springboot-example.jar

EXPOSE 8080

##### 配置容器启动后执行的命令
ENTRYPOINT ["java","-jar","/docker-springboot-example.jar"]
```

#### 打包jar并上传到Linux服务器中 ####

```bash
mvn clean package
```

![18328822-ad36-4b07-bda8-20f204e0a116.png][]

```bash
mkdir /usr/local/docker-app
cd /usr/local/docker-app
rz docker-springboot-example-0.0.1-SNAPSHOT.jar ### 上传jar文件
rz Dockerfile ### 上传Dockerfile文件
```

#### 构建镜像 ####

```bash
docker build -t docker-springboot-example .
### 语法： docker build -t 镜像名[:标签] Dockerfile所在位置
```

![b0a1a96d-3b81-4e9d-be21-aad4c7de19c5.png][]

#### 启动镜像 ####

```bash
docker run -p 8080:8080 docker-springboot-example
```

这里没有加`-d`参数，直接前台启动了，可以看到实时打印的控制台信息


[https_docs.docker.com]: https://docs.docker.com/
[http_www.docker.org.cn]: http://www.docker.org.cn/
[98beb935-9147-42b4-96c6-d6d4c5e10a56.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20181226/98beb935-9147-42b4-96c6-d6d4c5e10a56.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[cce73947-e8af-4e67-baf3-680749826e39.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20181226/cce73947-e8af-4e67-baf3-680749826e39.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[https_hub.docker.com_search_q_java_type_image]: https://hub.docker.com/search?q=java&type=image
[2019-08-19-73749]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20181226/e6a30999-1cb9-430e-9bcf-2a8b3f62c856.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[https_cr.console.aliyun.com_cn-hangzhou_mirrors]: https://cr.console.aliyun.com/cn-hangzhou/mirrors
[2019-08-19-073749]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20181226/c4e1f848-10b9-4f20-b75e-547bd792d93e.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[7fa2ab0b-9b83-4717-a0a2-0a7df5c811cc.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20181226/7fa2ab0b-9b83-4717-a0a2-0a7df5c811cc.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[148c41c3-0aa7-4629-a029-b54ea6637848.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20181226/148c41c3-0aa7-4629-a029-b54ea6637848.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[18328822-ad36-4b07-bda8-20f204e0a116.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20181226/18328822-ad36-4b07-bda8-20f204e0a116.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[b0a1a96d-3b81-4e9d-be21-aad4c7de19c5.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20181226/b0a1a96d-3b81-4e9d-be21-aad4c7de19c5.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg