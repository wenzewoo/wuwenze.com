+++
title = "使用Maven打包可执行JAR包"
date = "2018-08-29 06:51:00"
url = "archives/171"
tags = ["Maven"]
categories = ["后端"]
+++

### SpringBoot工程 ###

```xml
<plugin>
<groupId>org.springframework.boot</groupId>
<artifactId>spring-boot-maven-plugin</artifactId>
<configuration>
  <mainClass>com.wuwneze.springbootexample.SpringbootExampleApplication</mainClass>
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

### 普通Java工程 ###

```xml
<plugin>
<artifactId>maven-assembly-plugin</artifactId>
<configuration>
  <descriptorRefs>
    <descriptorRef>jar-with-dependencies</descriptorRef>
  </descriptorRefs>
  <archive>
    <manifest>
      <mainClass>com.wuwenze.Main</mainClass>
    </manifest>
  </archive>
</configuration>
<executions>
  <execution>
    <id>make-assembly</id>
    <phase>package</phase>
    <goals>
      <goal>single</goal>
    </goals>
  </execution>
</executions>
</plugin>
```

使用该插件生成的jar包，带jar-with-dependencies后缀的jar文件，可以直接运行。

### 如何打包 ###

```bash
mvn clean install package
```

### 题外话：如何让linux命令保持后台运行？ ###

```bash
nohup java -jar test-0.0.1-SNAPSHOT-jar-with-dependencies.jar >test.log 2>&1 &
```