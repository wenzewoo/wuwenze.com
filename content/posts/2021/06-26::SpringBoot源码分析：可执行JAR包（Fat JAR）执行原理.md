+++
title = "SpringBoot源码分析：可执行JAR包（Fat JAR）执行原理"
date = "2021-06-26 22:48:30"
url = "archives/793"
tags = ["Spring"]
categories = ["后端","SpringBoot源码阅读"]
featuredImage = "https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210923/6703e7d300e04452a499c5513e848ffc.png"
+++

SpringBoot颠覆了传统的web项目启动方式，初用时很是惊艳。与传统WAR包的不同之处在于，其打包的JAR包（Fat JAR）中内置了lib目录以及WEB容器，通过简单的`java -jar`命令即可启动WEB项目，大大的简化了项目的发布流程。

## Fat JAR包结构 ##

在此之前，咱们需要准备一个简单的SpringBoot项目，执行 `mvn clean package` 命令以后，得到两个文件：

 *  /target/springboot-loader-study-0.0.1-SNAPSHOT.jar
 *  /target/springboot-loader-study-0.0.1-SNAPSHOT.jar.original

前者为SpringBoot定制的`Fat JAR`，内部包含依赖的第三方jar包以及内置的Web容器，而后者则为原始的JAR包文件。

将一个JAR及其依赖的三方JAR全部打到一个包中，这个包即为Fat Jar，咱们解压出来看看：

![866deef5-398f-4a2d-a049-3fbba179d021.png][]

### 目录结构 ###

 *  `./：` SpringBoot类加载器
 *  `./META-INF/：`JAR包元信息
 *  `./BOOT-INF/classes/：`项目自身的classes文件
 *  `./BOOT-INF/lib/：`项目自身依赖的第三方JAR包文件
 *  `./BOOT-INF/lib/classpath.idx：`项目自身依赖的JAR包清单
 *  `./BOOT-INF/lib/layers.idx：`项目文件清单

打开`/META-INF/MANIFEST.MF`查看元信息

```bash
Manifest-Version: 1.0
Spring-Boot-Classpath-Index: BOOT-INF/classpath.idx
Implementation-Title: springboot-loader-study
Implementation-Version: 0.0.1-SNAPSHOT
Spring-Boot-Layers-Index: BOOT-INF/layers.idx
Start-Class: com.wuwenze.springbootloaderstudy.SpringbootLoaderStudyApplication
Spring-Boot-Classes: BOOT-INF/classes/
Spring-Boot-Lib: BOOT-INF/lib/
Build-Jdk-Spec: 1.8
Spring-Boot-Version: 2.5.1
Created-By: Maven Jar Plugin 3.2.0
Main-Class: org.springframework.boot.loader.JarLauncher
```

 *  **Main-Class**：org.springframework.boot.loader.JarLauncher
 *  **Start-Class**：com.wuwenze.springbootloaderstudy.SpringbootLoaderStudyApplication

`java -jar` 命令引导的启动类必须配置在 MANIFEST.MF 资源的 Main-Class 属性，从结构可以大致猜测出基本原理：

`org.springframework.boot.loader.JarLauncher`负责加载引导`/BOOT-INF`下的类以及第三方lib，最终调用`com.wuwenze.springbootloaderstudy.SpringbootLoaderStudyApplication`完成项目的启动

## JarLauncher 基本原理 ##

经过上面的简单分析，来思考一个问题，为什么SpringBoot要多此一举，为什么不直接通过`java -jar`来运行`SpringbootLoaderStudyApplication`启动类呢？

原因其实很简单，上面的图中咱们可以看到，要运行SpringBoot项目，除了项目本身的classes文件，还有一堆第三方的JAR包需要全部加载并放入到同一个`ClassLoader`下才可以成功运行，而原生的JAVA是做不到的，来看一段测试代码：

```java
public class JarLauncherTest {

    @Test // 测试原生URLClassLoader加载JAR包中的JAR包
    public void testJarlib() throws MalformedURLException, ClassNotFoundException {
        // 构建jar包本身的URL
        URL normalJarUrl = new URL("file:///Users/wuwenze/Public/Code/springboot-study/springboot-loader-study/target/springboot-loader-study-0.0.1-SNAPSHOT.jar");

        // 构建jar包中的/BOOT-INF/lib/spring-web-5.3.8.jar的URL
        URL jarUrl = new URL("jar:file:///Users/wuwenze/Public/Code/springboot-study/springboot-loader-study/target/springboot-loader-study-0.0.1-SNAPSHOT.jar!/BOOT-INF/lib/spring-web-5.3.8.jar!/");

        // 同时放入ClassLoader
        URLClassLoader urlClassLoader = new URLClassLoader(new URL[]{normalJarUrl, jarUrl}, null);

        // 加载类（JarLauncher类位于jar内部外层）
        System.out.println(urlClassLoader.loadClass("org.springframework.boot.loader.JarLauncher"));

        // 加载类（HttpStatus类位于jar包中的/BOOT-INF/lib/spring-web-5.3.8.jar）
        System.out.println(urlClassLoader.loadClass("org.springframework.http.HttpStatus"));
    }
}

// 执行结果：
class org.springframework.boot.loader.JarLauncher
java.lang.ClassNotFoundException: org.springframework.http.HttpStatus
```

由此可见，`org.springframework.http.HttpStatus`是加载不了的，如何解决呢？

 *  将所有的第三方lib，全部解压到到一起，与程序本身的classes放在一起，如此JAVA应用程序装载器）就能识别了（易冲突）
 *  使用JarLaunch，在启动时候将所有的的classes、第三方JAR放入ClassLoader。

![826a0b12-ee93-404e-be31-d2940fcde1a9.png][]

### URL解析原理 ###

要解决无法从JAR中加载JAR包的问题（即`URL=jar:file:xxx.jar!/xxx.jar`），首先需要简单了解一下JAVA中的URL解析原理。

![3996378c-b4a9-455a-a826-dcac895ff459.png][]

1.  解析URL对象，获得协议头（如：file）
2.  通过协议头，创建对应的`java.net.URLStreamHander`对象。
3.  获取对应的URLConnection，按需获取输入或者输出流。

如何获取到URLStreamHandler的实现类的呢？咱们来分析分析JAVA的源代码

```bash
// java.net.URL

private static final String protocolPathProp = "java.protocol.handler.pkgs";

// 省略若干代码...

static URLStreamHandler getURLStreamHandler(String protocol) {
    // 从本地缓存中读取
    URLStreamHandler handler = handlers.get(protocol);
    if (handler == null) {

        boolean checkedWithFactory = false;

        // 如果有URLStreamHandlerFactory工厂，则直接使用工厂构建
        if (factory != null) {
            handler = factory.createURLStreamHandler(protocol);
            checkedWithFactory = true;
        }

        // 如果没有，则基于JAVA包名进行匹配
        if (handler == null) {
            String packagePrefixList = null;
            
            // 从java环境变量中获取handler包路径前缀
            packagePrefixList
                = java.security.AccessController.doPrivileged(
                new sun.security.action.GetPropertyAction(
                    protocolPathProp,""));
            if (packagePrefixList != "") {
                packagePrefixList += "|";
            }
            // 追加默认的handler实现类前缀（多个以|分割）
            packagePrefixList += "sun.net.www.protocol";

            StringTokenizer packagePrefixIter =
                    new StringTokenizer(packagePrefixList, "|");

            while (handler == null &&
                    packagePrefixIter.hasMoreTokens()) {

                String packagePrefix =
                        packagePrefixIter.nextToken().trim();
                try {
                    // 即通过sun.net.www.protocol.协议名.Handler获取到对象，并反射其实例
                    String clsName = packagePrefix + "." + protocol + ".Handler";
                    Class<?> cls = null;
                    try {
                        cls = Class.forName(clsName);
                    } catch (ClassNotFoundException e) {
                        ClassLoader cl = ClassLoader.getSystemClassLoader();
                        if (cl != null) {
                            cls = cl.loadClass(clsName);
                        }
                    }
                    if (cls != null) {
                        handler  = (URLStreamHandler)cls.newInstance();
                    }
                } catch (Exception e) {
                    // any number of exceptions can get thrown here
                }
            }
        }

        // 省略若干代码...
    }

    return handler;

}
```

![59946655-bc82-4187-8357-a21371c2a89d.png][]图为URLStreamHandler默认的实现类

从源代码来分析，咱们了解到，获取URLStreamHandler有两种途径：

1.  使用`java.net.URL#setURLStreamHandlerFactory`工厂自行指定规则。
2.  使用JDK默认的规则匹配实现类名，并反射其实例，即：
    
    1.  `(环境变量java.protocol.handler.pkgs).{protocol}.Handler`
    2.  `sun.net.www.protocol.{protocol}.Handler`

### JarLauncher解决的问题 ###

分析JarLauncher源代码，SpringBoot采用了第二种方式实现自定义的URLStreamHandler来达到Fat JAR的功能。

```java
// org.springframework.boot.loader.JarLauncher

public class JarLauncher extends ExecutableArchiveLauncher {
    // 省略若干代码...
    public static void main(String[] args) throws Exception {
                // 调用自身的launch()函数（实则为父类Launcher.launch()）
        new JarLauncher().launch(args);
    }
}

// org.springframework.boot.loader.Launcher

public abstract class Launcher {

    private static final String JAR_MODE_LAUNCHER = "org.springframework.boot.loader.jarmode.JarModeLauncher";

    protected void launch(String[] args) throws Exception {
        if (!isExploded()) {
                        // 重新注册JAR URLStreamHandler
            JarFile.registerUrlProtocolHandler();
        }
        ClassLoader classLoader = createClassLoader(getClassPathArchivesIterator());
        String jarMode = System.getProperty("jarmode");
        String launchClass = (jarMode != null && !jarMode.isEmpty()) ? JAR_MODE_LAUNCHER : getMainClass();
        launch(args, launchClass, classLoader);
    }
}

// org.springframework.boot.loader.jar.JarFile

public class JarFile extends AbstractJarFile implements Iterable<java.util.jar.JarEntry> {

    private static final String PROTOCOL_HANDLER = "java.protocol.handler.pkgs";
    private static final String HANDLERS_PACKAGE = "org.springframework.boot.loader";


    // 省略若干代码...

    /**
     * Register a {@literal 'java.protocol.handler.pkgs'} property so that a
     * {@link URLStreamHandler} will be located to deal with jar URLs.
     */
    public static void registerUrlProtocolHandler() {
        Handler.captureJarContextUrl();

        // 追加环境变量java.protocol.handler.pkgs为Spring自定义的org.springframework.boot.loader
        // 那么jar包的URLStreamHandler即为org.springframework.boot.loader.jar.Handler
        String handlers = System.getProperty(PROTOCOL_HANDLER, "");
        System.setProperty(PROTOCOL_HANDLER,
                ((handlers == null || handlers.isEmpty()) ? HANDLERS_PACKAGE : handlers + "|" + HANDLERS_PACKAGE));
        resetCachedUrlHandlers(); // 重置缓存
    }

    // 省略若干代码...
}
```

这里再思考一个问题，为什么Spring选择了第二种看似不太优雅的方式来实现自定义的`Jar URLStreamHandler`呢？

咱们回到JDK的源码，继续研究一下`java.net.URL#setURLStreamHandlerFactory`的相关规则：

![e3c10457-9d1e-429d-a90a-9c29312c9057.png][]

通过源码发现，虽然可以通过URLStreamHandlerFactory工厂来实现，但是该工厂只能设置一次，故而在这里就不太合适了（WEB容器可能已经设置过了）

既然目前已经知道SpringBoot是如何替换默认的`jar.Handler`的了，那么咱们来改造一下刚才报错的单元测试，尝试让URLClassLoader加载JAR包中的第三方JAR文件内容试试看：

```xml
<!--引入spring-boot-loader，方便测试Spring自定义的JAR URLStreamHandler-->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-loader</artifactId>
    <version>2.5.1</version>
    <scope>test</scope>
</dependency>
```

```java
@Test
public void testJarlib2() throws MalformedURLException, ClassNotFoundException {
    // 设置自定义的URLStreamHandler实现类前缀
    final String protocolHandler = "java.protocol.handler.pkgs";
    final String handlersPackage = "org.springframework.boot.loader";
    String handlers = System.getProperty(protocolHandler, "");
    System.setProperty(protocolHandler,
            ((handlers == null || handlers.isEmpty()) ? handlersPackage : handlers + "|" + handlersPackage));

    // 构建jar包本身的URL
    URL normalJarUrl = new URL("file:///Users/wuwenze/Public/Code/springboot-study/springboot-loader-study/target/springboot-loader-study-0.0.1-SNAPSHOT.jar");

    // 构建jar包中的/BOOT-INF/lib/spring-web-5.3.8.jar的URL
    URL jarUrl = new URL("jar:file:///Users/wuwenze/Public/Code/springboot-study/springboot-loader-study/target/springboot-loader-study-0.0.1-SNAPSHOT.jar!/BOOT-INF/lib/spring-web-5.3.8.jar!/");

    // 同时放入ClassLoader
    URLClassLoader urlClassLoader = new URLClassLoader(new URL[]{normalJarUrl, jarUrl}, null);

    // 加载类（JarLauncher类位于jar内部外层）
    System.out.println(urlClassLoader.loadClass("org.springframework.boot.loader.JarLauncher"));

    // 加载类（HttpStatus类位于jar包中的/BOOT-INF/lib/spring-web-5.3.8.jar）
    System.out.println(urlClassLoader.loadClass("org.springframework.http.HttpStatus"));
}
```

![2db4f271-534f-4b62-9f84-e93494fd6d30.png][]

总算是成功了，完成了万里长征的第一步。

## JarLauncher引导过程详解 ##

在更深入的分析之前，咱们先来做一个简短的总结：

![45a0f204-2148-4c5f-b0e2-e108b937c503.png][]

`JarLauncher`继承于`org.springframework.boot.loader.ExecutableArchiveLauncher`。该类的无参构造方法最主要的功能就是构建了当前main方法所在的FatJar的`JarFileArchive`对象。

1.  以FatJar为file作为入参，构造JarFileArchive对象。获取其中所有的资源目标，取得其Url，将这些URL作为参数，构建了一个URLClassLoader。
2.  以第一步构建的ClassLoader加载`MANIFEST.MF`文件中`Start-Class`指向的业务类，并且通过反射执行静态方法main。进而启动整个程序。

通过静态方法`org.springframework.boot.loader.JarLauncher#main`()就可以顺利启动整个程序。这里面的关键在于SpringBoot自定义的ClassLoader能够识别FatJar中的资源，包括有：在指定目录下的项目编译class、在指令目录下的项目依赖JAR。JDK默认用于加载应用的AppClassLoader只能从jar的根目录开始加载class文件，并且若JAR中还内嵌有其他JAR，同样也不支持加载。

为了实现这个目标，SpringBoot首先从支持JAR包中的JAR包内容的读取这方面做了些定制：

1.  实现了一个`java.net.URLStreamHandler`的子类`org.springframework.boot.loader.jar.Handler`。该Handler支持识别多个`!/`分隔符，并且正确的打开`URLConnection`。打开的Connection是SpringBoot定制的`org.springframework.boot.loader.jar.JarURLConnection`实现。
2.  实现了一个`java.net.JarURLConnection`的子类`org.springframework.boot.loader.jar.JarURLConnection`。该链接支持多个`!/`分隔符，并且自己实现了在这种情况下获取InputStream的方法。而为了能够在`org.springframework.boot.loader.jar.JarURLConnection`正确获取输入流，SpringBoot自定义了一套读取ZipFile的工具类和方法。这部分和ZIP压缩算法规范紧密相连，过于复杂，就不再继续研究了。

实现以上的定制以后，剩下的事情就变得很简单了。构建好一个`JarFileArchive`后，通过该对象获取其内部所有的资源URL，这些URL包含项目编译class和依赖的第三方JAR包，将获取的URL数组作为参数传递给自定义的ClassLoader，即`org.springframework.boot.loader.LaunchedURLClassLoader`（继承自`URLClassLoader`）

至此，调用Start-Class将畅行无阻。

### 调试JarLauncher ###

为了更深入的理解JarLauncher的整体原理以及流程，咱们尝试打个断点，调试跟踪代码，借助于IDEA工具，直接调用`org.springframework.boot.loader.JarLauncher#main()`函数

```java
public static void main(String[] args) throws Exception {
    JarLauncher.main(args);
}
```

![44332562-c428-4cc9-a64e-0e0a939591c5.png][]

世事总是事与愿违，咱们迎来了第一个问题，在`MainMethodRunner`中尝试获取`Start-Class`反射调用的时候，出现了异常。

通过报错不难发现，再解析`METAINF`元信息时，并无法获取到`Start-Class`属性，原因很简单，咱们是直接通过Maven引入的`spring-boot-loader`进行调试，这里面当然没有我们需要的元信息，那么要如何解决呢？

1.  删除maven引入的用于测试的`spring-boot-loader`
2.  将springboot项目打包后生成的jar包，作为一个普通JAR包加载到项目中，再尝试进行调试

![93e899a2-57ac-4a3f-be94-7ba1b218ed04.png][]![daadfc21-a6ac-4e32-b2f8-874d88327367.png][]

图上所示，通过IDEA直接调用`JarLauncher.main()`已经成功的启动了SpringBoot项目。

由于时间和篇幅的原因，后续的代码调试与源码分析则不再细表了。


[866deef5-398f-4a2d-a049-3fbba179d021.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210626/866deef5-398f-4a2d-a049-3fbba179d021.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[826a0b12-ee93-404e-be31-d2940fcde1a9.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210626/826a0b12-ee93-404e-be31-d2940fcde1a9.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[3996378c-b4a9-455a-a826-dcac895ff459.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210626/3996378c-b4a9-455a-a826-dcac895ff459.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[59946655-bc82-4187-8357-a21371c2a89d.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210626/59946655-bc82-4187-8357-a21371c2a89d.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[e3c10457-9d1e-429d-a90a-9c29312c9057.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210626/e3c10457-9d1e-429d-a90a-9c29312c9057.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[2db4f271-534f-4b62-9f84-e93494fd6d30.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210626/2db4f271-534f-4b62-9f84-e93494fd6d30.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[45a0f204-2148-4c5f-b0e2-e108b937c503.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210626/45a0f204-2148-4c5f-b0e2-e108b937c503.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[44332562-c428-4cc9-a64e-0e0a939591c5.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210626/44332562-c428-4cc9-a64e-0e0a939591c5.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[93e899a2-57ac-4a3f-be94-7ba1b218ed04.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210626/93e899a2-57ac-4a3f-be94-7ba1b218ed04.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[daadfc21-a6ac-4e32-b2f8-874d88327367.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210626/daadfc21-a6ac-4e32-b2f8-874d88327367.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg