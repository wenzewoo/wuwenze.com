+++
title = "将JAR包发布到Maven中央仓库"
date = "2019-01-24 07:05:00"
url = "archives/229"
tags = ["Maven"]
categories = ["后端"]
+++

将JAR包发布到Maven中央仓库[https://search.maven.org][https_search.maven.org]供广大开发者使用，流程比较繁琐，遂成此文记录。

![9d814d09-314c-4acf-84df-08918f19a341.png][]

Maven中央仓库并不支持直接上传Jar包。因此需要将jar包发布到一些指定的第三方Maven仓库，然后该仓库再将Jar包同步到Maven中央仓库。

本文使用最简单的方式，通过发布到`Sonatype OSSRH` ([https://central.sonatype.org/pages/ossrh-guide.html][https_central.sonatype.org_pages_ossrh-guide.html]) 仓库的方式来实现。

### 注册JIRA账号 ###

JIRA是一个项目管理服务，类似于国内的Teambition。Sonatype通过JIRA来管理OSSRH仓库。

注册地址：[https://issues.sonatype.org/secure/Signup!default.jspa][https_issues.sonatype.org_secure_Signup_default.jspa]

需要填写Email, Full Name, Username以及password，其中Username与Password后面的步骤需要用到`（重要）`。

### 创建issue ###

在发布Jar包之前，需先创建issue，Sonatype的工作人员会进行审核。

申请地址：[https://issues.sonatype.org/secure/CreateIssue.jspa?issuetype=21&pid=10134][https_issues.sonatype.org_secure_CreateIssue.jspa_issuetype_21_pid_10134]

创建issue的时候需要填写下面这些信息：

 *  Summary：摘要
 *  Description：详细描述
 *  Group Id：对应Maven的GroupId
 *  Project URL：项目首页地址
 *  SCM url：项目Github地址

![b9781408-8581-49c6-99fa-80abe07a182f.png][]

在填写相关信息时，可以参照我之前创建的issue：[https://issues.sonatype.org/browse/OSSRH-38344][https_issues.sonatype.org_browse_OSSRH-38344]

由于时差问题，前一天创建issue，一般来说第二天早上才会有回应，工作人员会在回复中询问，是否拥有GroupId对应的域名所有权，

如果有的话就回复有，如果没有，建议使用github域名，如：com.github.wuwz：

![cde70e51-fc77-42a4-a8f2-c96baef4d570.png][]

当issue的status变为RESOLVED，我们就可以进行下一步操作了。

### GPG配置与安装 ###

为防止上传的Jar包被纂改，发布到Maven仓库中的所有文件都需要使用GPG签名。

#### 安装GPG ####

 *  Windows：下载Gpg4win（[https://www.gpg4win.org/download.html）][https_www.gpg4win.org_download.html]
 *  MacOS: 下载GPG Suite（[https://gpgtools.org/）][https_gpgtools.org]
 *  Linux：yum install gpg

#### 生成GPG密钥对 ####

```bash
gpg --gen-key

gpg (GnuPG) 2.2.11-unknown; Copyright (C) 2018 Free Software Foundation, Inc.
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.

gpg: directory '/c/Users/info016/.gnupg' created
gpg: keybox '/c/Users/info016/.gnupg/pubring.kbx' created
Note: Use "gpg --full-generate-key" for a full featured key generation dialog.

GnuPG needs to construct a user ID to identify your key.

Real name: wuwenze
Email address: wenzewoo@gmail.com
You selected this USER-ID:
    "wuwenze <wenzewoo@gmail.com>"

Change (N)ame, (E)mail, or (O)kay/(Q)uit? o
We need to generate a lot of random bytes. It is a good idea to perform
some other action (type on the keyboard, move the mouse, utilize the
disks) during the prime generation; this gives the random number
generator a better chance to gain enough entropy.
We need to generate a lot of random bytes. It is a good idea to perform
some other action (type on the keyboard, move the mouse, utilize the
disks) during the prime generation; this gives the random number
generator a better chance to gain enough entropy.
gpg: /c/Users/info016/.gnupg/trustdb.gpg: trustdb created
gpg: key D271764618CA9BBC marked as ultimately trusted
gpg: directory '/c/Users/info016/.gnupg/openpgp-revocs.d' created
gpg: revocation certificate stored as '/c/Users/info016/.gnupg/openpgp-revocs.d/AD320C934CC0DCA8C27C4407D271764618CA9BBC.rev'
public and secret key created and signed.

pub   rsa2048 2019-01-24 [SC] [expires: 2021-01-23]
      AD320C934CC0DCA8C27C4407D271764618CA9BBC
uid                      wuwenze <wenzewoo@gmail.com>
sub   rsa2048 2019-01-24 [E] [expires: 2021-01-23]
```

生成密钥时将需要输入name、email以及password。password在之后的步骤需要用到，请记下来。

#### 上传GPG公钥 ####

将公钥上传到公共的密钥服务器，这样其他人才可以通过公钥来验证jar包的完整性。

```bash
gpg --keyserver hkp://keyserver.ubuntu.com:11371 --send-keys AD320C934CC0DCA8C27C4407D271764618CA9BBC
```

其中，`AD320C934CC0DCA8C27C4407D271764618CA9BBC`为秘钥的ID，可以通过`gpg --list-keys`命令来查看：

```bash
gpg --list-keys
gpg: checking the trustdb
gpg: marginals needed: 3  completes needed: 1  trust model: pgp
gpg: depth: 0  valid:   1  signed:   0  trust: 0-, 0q, 0n, 0m, 0f, 1u
gpg: next trustdb check due at 2021-01-23
/c/Users/info016/.gnupg/pubring.kbx
-----------------------------------
pub   rsa2048 2019-01-24 [SC] [expires: 2021-01-23]
      AD320C934CC0DCA8C27C4407D271764618CA9BBC
uid           [ultimate] wuwenze <wenzewoo@gmail.com>
sub   rsa2048 2019-01-24 [E] [expires: 2021-01-23]
```

### Maven全局setting.xml配置 ###

注意，此处是全局的setting.xml，例如我的路径是：`D:Mavenconfsettings.xml`，添加以下内容：

```xml
<servers>
  <server>
    <id>sonatype</id>
    <username>注册的JIRA账号</username>
    <password>注册的JIRA密码</password>
  </server>
</servers>
```

### Maven项目的pom.xml配置 ###

根据Sonatype OSSRH的要求，以下信息都必须配置：

 *  Supply Javadoc and Sources
 *  Sign Files with GPG/PGP
 *  Sufficient Metadata
    
     *  Correct Coordinates
     *  Project Name, Description and URL
     *  License Information
     *  Developer Information
     *  SCM Information

配置挺多的，配置时参考我的项目，然后进行修改即可：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <groupId>com.wuwenze</groupId>
  <artifactId>ExcelKit</artifactId>
  <version>2.0.7</version>
  <packaging>jar</packaging>
  <name>ExcelKit</name>
  <url>http://gitee.com/wuwenze/ExcelKit</url>
  <description>Excel导入导出工具（简单、好用且轻量级的海量Excel文件导入导出解决方案.）</description>
  <developers>
    <developer>
      <name>wuwenze</name>
      <url>https://wuwenze.com</url>
      <email>wenzewoo@gmail.com</email>
    </developer>
  </developers>
  <licenses>
    <license>
      <name>The Apache Software License, Version 2.0</name>
      <url>http://www.apache.org/licenses/LICENSE-2.0.txt</url>
    </license>
  </licenses>
  <scm>
    <connection>scm:git:git@gitee.com:wuwenze/ExcelKit.git</connection>
    <developerConnection>scm:git:git@gitee.com:wuwenze/ExcelKit.git</developerConnection>
    <url>git@gitee.com:wuwenze/ExcelKit.git</url>
  </scm>

  <properties>
    <encoding>UTF-8</encoding>
    <jdk-version>1.6</jdk-version>
    <poi-version>3.17</poi-version>
    <dom4j-version>1.6.1</dom4j-version>
    <jaxen-version>1.1.6</jaxen-version>
    <xerces-version>2.11.0</xerces-version>
    <guava-version>18.0</guava-version>
  </properties>

  <dependencies>
    <dependency>
      <groupId>org.apache.poi</groupId>
      <artifactId>poi-ooxml</artifactId>
      <version>${poi-version}</version>
    </dependency>
    <dependency>
      <groupId>org.apache.poi</groupId>
      <artifactId>poi-ooxml-schemas</artifactId>
      <version>${poi-version}</version>
    </dependency>
    <dependency>
      <groupId>dom4j</groupId>
      <artifactId>dom4j</artifactId>
      <version>${dom4j-version}</version>
    </dependency>
    <dependency>
      <groupId>jaxen</groupId>
      <artifactId>jaxen</artifactId>
      <version>${jaxen-version}</version>
    </dependency>
    <dependency>
      <groupId>xerces</groupId>
      <artifactId>xercesImpl</artifactId>
      <version>${xerces-version}</version>
    </dependency>
    <dependency>
      <groupId>xml-apis</groupId>
      <artifactId>xml-apis</artifactId>
      <version>2.0.2</version>
    </dependency>
    <dependency>
      <groupId>com.google.guava</groupId>
      <artifactId>guava</artifactId>
      <version>18.0</version>
    </dependency>
    <dependency>
      <groupId>org.projectlombok</groupId>
      <artifactId>lombok</artifactId>
      <version>1.16.10</version>
      <scope>provided</scope>
    </dependency>
    <dependency>
      <groupId>javax.servlet</groupId>
      <artifactId>servlet-api</artifactId>
      <version>2.5</version>
      <scope>provided</scope>
    </dependency>
    <dependency>
      <groupId>commons-beanutils</groupId>
      <artifactId>commons-beanutils</artifactId>
      <version>1.9.3</version>
    </dependency>
    <dependency>
      <groupId>junit</groupId>
      <artifactId>junit</artifactId>
      <version>4.12</version>
      <scope>test</scope>
    </dependency>
  </dependencies>

  <build>
    <plugins>
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-compiler-plugin</artifactId>
        <configuration>
          <target>${jdk-version}</target>
          <source>${jdk-version}</source>
          <encoding>${encoding}</encoding>
        </configuration>
      </plugin>
    </plugins>
  </build>

  <profiles>
    <profile>
      <id>jdk-profile</id>
      <activation>
        <activeByDefault>true</activeByDefault>
        <jdk>${jdk-version}</jdk>
      </activation>
      <properties>
        <maven.compiler.source>${jdk-version}</maven.compiler.source>
        <maven.compiler.target>${jdk-version}</maven.compiler.target>
        <maven.compiler.compilerVersion>${jdk-version}</maven.compiler.compilerVersion>
      </properties>
    </profile>
    <profile>
      <id>release</id>
      <build>
        <plugins>
          <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-source-plugin</artifactId>
            <version>2.2.1</version>
            <executions>
              <execution>
                <phase>package</phase>
                <goals>
                  <goal>jar-no-fork</goal>
                </goals>
              </execution>
            </executions>
          </plugin>
          <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-javadoc-plugin</artifactId>
            <version>2.9.1</version>
            <executions>
              <execution>
                <phase>package</phase>
                <goals>
                  <goal>jar</goal>
                </goals>
              </execution>
            </executions>
          </plugin>
          <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-gpg-plugin</artifactId>
            <version>1.5</version>
            <executions>
              <execution>
                <phase>verify</phase>
                <goals>
                  <goal>sign</goal>
                </goals>
              </execution>
            </executions>
          </plugin>
        </plugins>
      </build>
      <distributionManagement>
        <snapshotRepository>
          <id>sonatype</id>
          <url>https://oss.sonatype.org/content/repositories/snapshots/</url>
        </snapshotRepository>
        <repository>
          <id>sonatype</id>
          <url>https://oss.sonatype.org/service/local/staging/deploy/maven2/</url>
        </repository>
      </distributionManagement>
    </profile>
  </profiles>

</project>
```

### Deploy Jar ###

前面的准备工作一切妥当后，即可执行deploy操作了，执行以下Maven命令：

```bash
mvn clean package deploy -Prelease
```

![bbdef5df-ede0-4c77-bb70-46b4cf6f9c89.png][]

第一次执行该命令时，需要输入GPG的密码，之前配置的时候已经记录过了，然后静静的等待上传成功（这其中有很多坑，自己根据Maven报错依次解决，直到最终SUCCESS就行了，这里不表。）

### Release Jar ###

使用JIRA账号登陆：[https://oss.sonatype.org/\#stagingRepositories，简短的说一下操作步骤吧：][https_oss.sonatype.org_stagingRepositories]

 *  将Staging Rpositories拉到最下即可看到你刚刚发布的jar包。

![a208eb31-57a4-4053-a421-4043282d83ec.png][]

 *  选择上方的Close，第一次会有工作人员回复你之前创建的那个Issue（前面有截图）。
 *  然后再点击Release即可，等2个小时左右即可在[http://search.maven.org][http_search.maven.org]看到你发布的jar包

![49ca95f5-7567-4bff-99a4-f1aa5f634690.png][]

 *  注意：在流程中可能失败，这时可以点击Activity查看具体的错误原因：

![e0a95e64-84b7-4ca6-b5e6-3c233f18ec09.png][]

### Update Jar ###

第一次相对来说比较麻烦，以后更新jar包就比较简单了，更改pom.xml中的version，重新deploy、close、release流程即可。


[https_search.maven.org]: https://search.maven.org
[9d814d09-314c-4acf-84df-08918f19a341.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20190124/9d814d09-314c-4acf-84df-08918f19a341.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[https_central.sonatype.org_pages_ossrh-guide.html]: https://central.sonatype.org/pages/ossrh-guide.html
[https_issues.sonatype.org_secure_Signup_default.jspa]: https://issues.sonatype.org/secure/Signup!default.jspa
[https_issues.sonatype.org_secure_CreateIssue.jspa_issuetype_21_pid_10134]: https://issues.sonatype.org/secure/CreateIssue.jspa?issuetype=21&pid=10134
[b9781408-8581-49c6-99fa-80abe07a182f.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20190124/b9781408-8581-49c6-99fa-80abe07a182f.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[https_issues.sonatype.org_browse_OSSRH-38344]: https://issues.sonatype.org/browse/OSSRH-38344
[cde70e51-fc77-42a4-a8f2-c96baef4d570.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20190124/cde70e51-fc77-42a4-a8f2-c96baef4d570.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[https_www.gpg4win.org_download.html]: https://www.gpg4win.org/download.html%EF%BC%89
[https_gpgtools.org]: https://gpgtools.org/%EF%BC%89
[bbdef5df-ede0-4c77-bb70-46b4cf6f9c89.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20190124/bbdef5df-ede0-4c77-bb70-46b4cf6f9c89.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[https_oss.sonatype.org_stagingRepositories]: https://oss.sonatype.org/#stagingRepositories%EF%BC%8C%E7%AE%80%E7%9F%AD%E7%9A%84%E8%AF%B4%E4%B8%80%E4%B8%8B%E6%93%8D%E4%BD%9C%E6%AD%A5%E9%AA%A4%E5%90%A7%EF%BC%9A
[a208eb31-57a4-4053-a421-4043282d83ec.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20190124/a208eb31-57a4-4053-a421-4043282d83ec.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[http_search.maven.org]: http://search.maven.org
[49ca95f5-7567-4bff-99a4-f1aa5f634690.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20190124/49ca95f5-7567-4bff-99a4-f1aa5f634690.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[e0a95e64-84b7-4ca6-b5e6-3c233f18ec09.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20190124/e0a95e64-84b7-4ca6-b5e6-3c233f18ec09.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg