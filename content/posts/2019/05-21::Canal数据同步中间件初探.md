+++
title = "Canal数据同步中间件初探"
date = "2019-05-21 16:31:00"
url = "archives/597"
tags = ["MySQL","Canal","数据同步"]
categories = ["后端"]
+++

`MySQL`本身是支持主从模式（`Master`/`Slave`）的，`Master`产生的日志（`binary log`）中记录了所有增删改语句，将日志发送到`Slave`执行即可完成数据库的增量数据同步操作。

[Canal][]是阿里巴巴开源的一个中间件，他的作用就是解析`binary log`来完成数据同步的。  
源码地址：[https://github.com/alibaba/canal][Canal]

### Canal工作原理 ###

![2019-08-19-73552][]

1.  `canal`模拟`mysql slave`的交互协议，伪装自己为`mysql slave`，向`mysql master`发送`dump`协议
2.  `mysql master`收到`dump`请求，开始推送`binary log`给`slave`(也就是canal)
3.  `canal`解析`binary log`对象(原始为byte流)

### Canal Server搭建 ###

下载最新的`Releases`包，地址：[https://github.com/alibaba/canal/releases][https_github.com_alibaba_canal_releases]

```bash
wget https://github.com/alibaba/canal/releases/download/canal-1.1.3/canal.deployer-1.1.3.tar.gz
```

解压文件包到当前目录

```bash
tar -zxvf canal.deployer-1.1.3.tar.gz
```

![2019-08-19-073554][]

编辑配置文件：`conf/example/instance.properties`

```properties
##################################################
#### mysql serverId , v1.0.26+ will autoGen
### canal.instance.mysql.slaveId=0

### enable gtid use true/false
canal.instance.gtidon=false

### position info
canal.instance.master.address=localhost:3306
canal.instance.master.journal.name=
canal.instance.master.position=
canal.instance.master.timestamp=
canal.instance.master.gtid=

### rds oss binlog
canal.instance.rds.accesskey=
canal.instance.rds.secretkey=
canal.instance.rds.instanceId=

### table meta tsdb info
canal.instance.tsdb.enable=true
##canal.instance.tsdb.url=jdbc:mysql://127.0.0.1:3306/canal_tsdb
##canal.instance.tsdb.dbUsername=canal
##canal.instance.tsdb.dbPassword=canal

##canal.instance.standby.address =
##canal.instance.standby.journal.name =
##canal.instance.standby.position =
##canal.instance.standby.timestamp =
##canal.instance.standby.gtid=

### username/password
canal.instance.dbUsername=canal
canal.instance.dbPassword=canal
canal.instance.defaultDatabaseName = testcanal
canal.instance.connectionCharset = UTF-8
### enable druid Decrypt database password
canal.instance.enableDruid=false
##canal.instance.pwdPublicKey=MFwwDQYJKoZIhvcNAQEBBQADSwAwSAJBALK4BUxdDltRRE5/zXpVEVPUgunvscYFtEip3pmLlhrWpacX7y7GCMo2/JM6LeHmiiNdH1FWgGCpUfircSwlWKUCAwEAAQ==

### table regex
canal.instance.filter.regex=.*\..*
### table black regex
canal.instance.filter.black.regex=

### mq config
canal.mq.topic=example
### dynamic topic route by schema or table regex
##canal.mq.dynamicTopic=mytest1.user,mytest2\..*,.*\..*
canal.mq.partition=0
### hash partition config
##canal.mq.partitionsNum=3
##canal.mq.partitionHash=test.table:id^name,.*\..*
##################################################
```

修改`canal.instance.master.address`为要监听的MySQL连接地址：`localhost:3306`  
修改`canal.instance.defaultDatabaseName`为要监听的MySQL数据库名称：`testcanal`  
`canal.instance.dbUsername `和 `canal.instance.dbPassword`为数据库连接的用户名，用默认的就好，需要先创建对应的MySQL用户 （必须要`SLAVE权限`）。

```sql
CREATE USER canal IDENTIFIED BY 'canal';  
GRANT SELECT, REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'canal'@'%';
-- GRANT ALL PRIVILEGES ON *.* TO 'canal'@'%' ;
FLUSH PRIVILEGES;
```

此外，canal的原理是基于`mysql binlog`技术，所以一定需要开启mysql的binlog写入功能：

```bash
[mysqld]
log-bin=mysql-bin ##添加这一行就ok
binlog-format=ROW ##选择row模式
server_id=1 ##配置mysql replaction需要定义，不能和canal的slaveId重复
```

需要注意的是，`MySQL`高版本已经没有了`my.ini`配置文件，需要用户自己创建  
![2019-08-19-73555][]

**如果你使用的是阿里云RDS，账号默认已经有binlog dump权限，可以直接跳过这一步。**

启动服务：

```bash
./bin/startup.sh
```

查看日志：

```bash
tail -f logs/canal/canal.log
```

查看具体instance的日志:

```bash
tail -f logs/example/example.log
```

![2019-08-19-073556][]

### Canal Client编写 ###

服务端启动完毕后，可以在Client监听到指定库的变化，这里需要自己写客户端，将触发的变化打印出来，后续的业务逻辑就自行处理了。

创建一个标准的`Gradle`项目，这里使用`Kotlin`来编写相关的处理逻辑。（java大同小异）

```kotlin
import org.jetbrains.kotlin.gradle.tasks.KotlinCompile

plugins {
    kotlin("jvm") version "1.3.31"
}

group = "com.wuwenze"
version = "1.0-SNAPSHOT"

repositories {
    mavenCentral()
}

dependencies {
    implementation(kotlin("stdlib-jdk8"))
    implementation("com.alibaba.otter:canal.client:1.1.0")
}

tasks.withType<KotlinCompile> {
    kotlinOptions.jvmTarget = "1.8"
}
```

编写一个简单的监听程序：

```kotlin
import com.alibaba.otter.canal.client.CanalConnectors
import com.alibaba.otter.canal.common.utils.AddressUtils
import com.alibaba.otter.canal.protocol.CanalEntry
import com.alibaba.otter.canal.protocol.Message
import java.net.InetSocketAddress

/** * @author wuwenze * @date 2019-05-21 */
fun main(args: Array<String>) {
    // 创建连接，相关的端口，用户名在配置文件conf/canal.properties中
    val connector = CanalConnectors.newSingleConnector(
        InetSocketAddress(
            AddressUtils.getHostIp(),
            11111
        ), "example", "", ""
    )
    try {
        // 连接 & 订阅
        connector.connect()
        connector.subscribe(".*\..*")

        // 持续从Canal Server获取消息
        while (true) {
            // 获取指定数量的数据
            val message = connector.getWithoutAck(1000)

            // 调用自定义消息处理程序
            message.processingMessage()

            // 提交确认
            connector.ack(message.id)
        }
    } finally {
        connector.disconnect()
    }
}

private fun Message.processingMessage() {
    // 未接收到消息，休眠1秒
    if (this.entries.isEmpty()) {
        Thread.sleep(1000)
        return
    }

    this.entries.forEach {
        // 忽略事物消息
        if (it.entryType == CanalEntry.EntryType.TRANSACTIONBEGIN || it.entryType == CanalEntry.EntryType.TRANSACTIONEND) {
            return@forEach
        }

        // 解析消息
        val rowChange: CanalEntry.RowChange
        try {
            rowChange = CanalEntry.RowChange.parseFrom(it.storeValue)
        } catch (e: java.lang.Exception) {
            throw RuntimeException("ERROR #### parser of eromanga-event has an error , data:$it", e)
        }

        println("> [${rowChange.eventType}] name[${it.header.schemaName},${it.header.tableName}]")

        // 打印SQL语句
        if (rowChange.hasSql()) {
            println("SQL: ${rowChange.sql}")
        }

        // 打印DML信息
        rowChange.rowDatasList.forEach { rowData ->
            println("Before:")
            printColumn(rowData.beforeColumnsList)

            println("After:")
            printColumn(rowData.afterColumnsList)
        }

        // 事件类型处理分支
        when(rowChange.eventType) {
            CanalEntry.EventType.INSERT -> println("新增事件")
            CanalEntry.EventType.DELETE -> println("删除事件")
            // @see https://github.com/alibaba/canal/blob/master/protocol/src/main/java/com/alibaba/otter/canal/protocol/CanalEntry.java
            else -> {
                println("其他事件[${rowChange.eventType}]")
            }
        }
    }
}

private fun printColumn(columns: List<CanalEntry.Column>) {
    columns.forEach {
        println("column.name: ${it.name}, column.value: ${it.value}, column.updated: ${it.updated}")
    }
}
```

启动程序，开始接收`Canal Server`监听到的binlog消息。

### 测试 ###

修改相关的数据，在`Canal Client`控制台会收到相关的消息。

```sql
### 添加表
CREATE TABLE table1
(
    id    INT AUTO_INCREMENT,
    value VARCHAR(20) NOT NULL,
    CONSTRAINT table1_pk
        PRIMARY KEY (id)
)
    COMMENT '测试表1';

### 修改表
ALTER TABLE table1
    MODIFY value VARCHAR(255) NOT NULL;

### 加入数据
INSERT INTO table1(value)
VALUES ('test1');
INSERT INTO table1(value)
VALUES ('test2');

### 修改数据
UPDATE table1 t
SET t.value = 'test1_updated'
WHERE t.id = 1;

### 删除表
DROP TABLE table1;
```

`Canal Client`收到的完整消息如下：

```bash
> [CREATE] name[testcanal,table1]
SQL: /* ApplicationName=DataGrip 2019.1.2 */ CREATE TABLE table1
(
    id    INT AUTO_INCREMENT,
    value VARCHAR(20) NOT NULL,
    CONSTRAINT table1_pk
        PRIMARY KEY (id)
)
    COMMENT '测试表1'
其他事件[CREATE]
> [ALTER] name[testcanal,table1]
SQL: /* ApplicationName=DataGrip 2019.1.2 */ ALTER TABLE table1
    MODIFY value VARCHAR(255) NOT NULL
其他事件[ALTER]
> [INSERT] name[testcanal,table1]
Before:
After:
column.name: id, column.value: 1, column.updated: true
column.name: value, column.value: test1, column.updated: true
新增事件
> [INSERT] name[testcanal,table1]
Before:
After:
column.name: id, column.value: 2, column.updated: true
column.name: value, column.value: test2, column.updated: true
新增事件
> [UPDATE] name[testcanal,table1]
Before:
column.name: id, column.value: 1, column.updated: false
column.name: value, column.value: test1, column.updated: false
After:
column.name: id, column.value: 1, column.updated: false
column.name: value, column.value: test1_updated, column.updated: true
其他事件[UPDATE]
> [ERASE] name[testcanal,table1]
SQL: DROP TABLE `table1` /* generated by server */
其他事件[ERASE]
```

![2019-08-19-073558][]

### 其他 ###

更多例子，参考官方Wiki：[https://github.com/alibaba/canal/wiki][https_github.com_alibaba_canal_wiki]


[Canal]: https://github.com/alibaba/canal
[2019-08-19-73552]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20190521/0d73881b-c21f-4bca-afec-0c910676ac49.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[https_github.com_alibaba_canal_releases]: https://github.com/alibaba/canal/releases
[2019-08-19-073554]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20190521/b1ffefd0-afcc-40f2-8225-d30b84f6a86d.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[2019-08-19-73555]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20190521/a6666853-f708-4c22-962b-bba7bee5f616.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[2019-08-19-073556]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20190521/319e8d3c-6963-4419-a6a1-1d000c786bee.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[2019-08-19-073558]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20190521/3855d35f-c9b9-481a-bb77-e42c93eaff5d.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[https_github.com_alibaba_canal_wiki]: https://github.com/alibaba/canal/wiki