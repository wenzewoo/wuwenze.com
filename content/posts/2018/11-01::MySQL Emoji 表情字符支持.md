+++
title = "MySQL Emoji 表情字符支持"
date = "2018-11-01 07:59:00"
url = "archives/263"
tags = ["MySQL","Emoji"]
categories = ["后端"]
+++

utf-8编码可能2个字节、3个字节、4个字节的字符，但是MySQL的utf8编码只支持3字节的数据，而移动端的表情数据是4个字节的字符。如果直接往采用utf-8编码的数据库中插入表情数据，Java程序中将报SQL异常：

```bash
java.sql.SQLException: Incorrect string value: ‘xF0x9Fx92x94’ for column ‘name’ at row 1
```

针对该问题处理最简单的方式是更改数据库的编码为utf8mb4，直接将问题扼杀在摇篮中。

### 前置条件 ###

 *  MySQL服务器版本不能低于5.5.3;
 *  MySQL驱动版本不能低于5.1.13;

### 配置文件调整(my.cnf) ###

```bash
[client] 
default-character-set = utf8mb4 
[mysql] 
default-character-set = utf8mb4 
[mysqld] 
character-set-client-handshake = FALSE 
character-set-server = utf8mb4 
collation-server = utf8mb4_general_ci 
init_connect='SET NAMES utf8mb4'
```

重启数据库。

### 检查数据库变量 ###

```bash
SHOW VARIABLES WHERE Variable_name LIKE 'character_set_%' OR Variable_name LIKE 'collation%';
```

| Variable_name            | Value              |
|--------------------------|--------------------|
| character_set_client     | utf8mb4            |
| character_set_connection | utf8mb4            |
| character_set_database   | utf8mb4            |
| character_set_filesystem | binary             |
| character_set_results    | utf8mb4            |
| character_set_server     | utf8mb4            |
| character_set_system     | utf8mb4            |
| collation_connection     | utf8mb4_general_ci |
| collation_database       | utf8mb4_general_ci |
| collation_server         | utf8mb4_general_ci |


### 更改现有数据库、表、字段编码 ###

```sql
ALTER DATABASE `ticketdb` CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci;
ALTER TABLE `ticket_description` CONVERT TO CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci;
ALTER TABLE `ticket_description`
MODIFY COLUMN `description`  text CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL COMMENT '工单描述' AFTER `id`;
```

如此一来，emoji表情应该不是问题了。