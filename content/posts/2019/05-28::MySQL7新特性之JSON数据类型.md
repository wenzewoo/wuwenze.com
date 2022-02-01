+++
title = "MySQL7新特性之JSON数据类型"
date = "2019-05-28 16:34:00"
url = "archives/606"
tags = ["MySQL"]
categories = ["后端"]
+++

从`MySQL 5.7.8`开始，原生提供了一个JSON类型的数据格式，在此之前类似的需求都是需要通过VARCHAR的方式来存储处理的。

 *  JSON数据类型，拥有自动校验格式功能；
 *  提供操作JSON数据的内置函数；
 *  优化的存储格式，存储在JSON列中的JSON数据被转换成内部的存储格式，允许快速读取；
 *  支持修改JSON对象的特定属性，而不需要更新整个JSON内容；
 *  支持创建JSON对象的特定属性索引；

### 创建表 ###

```sql
CREATE TABLE user
(
    id       INT AUTO_INCREMENT,
    username VARCHAR(12) NOT NULL COMMENT '用户名',
    password VARCHAR(32) NOT NULL COMMENT '密码',
    extends  JSON        NULL COMMENT '扩展信息',
    CONSTRAINT user_pk
        PRIMARY KEY (id)
)
    COMMENT '用户表';

CREATE UNIQUE INDEX user_username_uindex
    ON user (username);
```

直接指定字段的类型为JSON即可。

### 初始化数据 ###

```bash
sql> INSERT INTO user (username, password, extends)
     VALUES ('admin', '123456', '{
       "nickname": "管理员",
       "age": 19,
       "open_id": "xxxxxxxxxxx"
     }')
[2019-05-27 18:37:12] 1 row affected in 14 ms
sql> INSERT INTO user (username, password, extends)
     VALUES ('guest', '123456', '{
       "nickname": "访客",
       "age": 12,
       "open_id": "yyyyyyyyyyyy"
     }')
[2019-05-27 18:37:12] 1 row affected in 5 ms
```

直接插入JSON格式的字符串，如果插入的JSON格式不正确，还会抛出异常：  
![2019-08-19-073241][]

表中的数据如下：

```bash
mysql> SELECT * FROM user;
+----+----------+----------+--------------------------------------------------------------------+
| id | username | password | extends                                                            |
+----+----------+----------+--------------------------------------------------------------------+
|  1 | admin    | 123456   | {"age": 19, "open_id": "xxxxxxxxxxx", "nickname": "管理员"}        |
|  2 | guest    | 123456   | {"age": 12, "open_id": "yyyyyyyyyyyy", "nickname": "访客"}         |
+----+----------+----------+--------------------------------------------------------------------+
2 rows in set (0.00 sec)
```

### 相关函数 ###

| Name                 | Description                                                                                                               |
|----------------------|---------------------------------------------------------------------------------------------------------------------------|
| JSON_APPEND()        | Append data to JSON document                                                                                              |
| JSON_ARRAY()         | Create JSON array                                                                                                         |
| JSON_ARRAY_APPEND()  | Append data to JSON document                                                                                              |
| JSON_ARRAY_INSERT()  | Insert into JSON array                                                                                                    |
| ->                   | Return value from JSON column after evaluating path; equivalent to JSON_EXTRACT().                                        |
| JSON_CONTAINS()      | Whether JSON document contains specific object at path                                                                    |
| JSON_CONTAINS_PATH() | Whether JSON document contains any data at path                                                                           |
| JSON_DEPTH()         | Maximum depth of JSON document                                                                                            |
| JSON_EXTRACT()       | Return data from JSON document                                                                                            |
| ->>                  | Return value from JSON column after evaluating path and unquoting the result; equivalent to JSON_UNQUOTE(JSON_EXTRACT()). |
| JSON_INSERT()        | Insert data into JSON document                                                                                            |
| JSON_KEYS()          | Array of keys from JSON document                                                                                          |
| JSON_LENGTH()        | Number of elements in JSON document                                                                                       |
| JSON_MERGE()         | Merge JSON documents                                                                                                      |
| JSON_OBJECT()        | Create JSON object                                                                                                        |
| JSON_QUOTE()         | Quote JSON document                                                                                                       |
| JSON_REMOVE()        | Remove data from JSON document                                                                                            |
| JSON_REPLACE()       | Replace values in JSON document                                                                                           |
| JSON_SEARCH()        | Path to value within JSON document                                                                                        |
| JSON_SET()           | Insert data into JSON document                                                                                            |
| JSON_TYPE()          | Type of JSON value                                                                                                        |
| JSON_UNQUOTE()       | Unquote JSON value                                                                                                        |
| JSON_VALID()         | Whether JSON value is valid                                                                                               |


### JSON取值表达式 ###

表示整个json对象，在索引数据时用下标(对于jsonarray，从0开始)或键值(对于jsonobject，含有特殊字符的key要用"括起来，比如表示整个json对象，在索引数据时用下标(对于jsonarray，从0开始)或键值(对于jsonobject，含有特殊字符的key要用"括起来，比如."my name")。

下面为一个相对完整的例子：

```bash
[3, {"a": [5, 6], "b": 10}, [99, 100]]

$[0] = 3
$[1] = {"a": [5, 6], "b": 10}
$[2] = [99, 100]
$[3] = NULL
$[1].a = [5, 6]
$[1].a[1] = 6
$[1].b = 10
$[2][0] = 99
```

### 常用函数示例 ###

下面列举一些常用的函数：

#### JSON\_ARRAY ####

> 生成一个包含指定元素的json数组。

```bash
mysql> SELECT JSON_ARRAY(1,"2","CCC", TRUE, CURTIME()) FROM DUAL;
+------------------------------------------+
| JSON_ARRAY(1,"2","CCC", TRUE, CURTIME()) |
+------------------------------------------+
| [1, "2", "CCC", true, "09:17:22.000000"] |
+------------------------------------------+
1 row in set (0.00 sec)
```

#### JSON\_OBJECT ####

> 生成一个包含指定K-V对的json object。如果有key为NULL或参数个数为奇数，则抛错。

```bash
mysql> INSERT INTO user (username, password, extends)
    -> VALUES ('test', '123123', JSON_OBJECT('nickname', '测试用户', 'age', 12, 'open_id', 'bbbbbbbbbbbb'))
    -> ;
Query OK, 1 row affected (0.00 sec)
```

![2019-08-19-073242][]

#### JSON\_CONTAINS ####

> 查询json文档是否在指定path包含指定的数据，包含则返回1，否则返回0。如果有参数为NULL或path不存在，则返回NULL。

```bash
mysql> SELECT t.username, JSON_CONTAINS(t.extends, '12', '$.age') AS ageIs12 FROM user t;
+----------+---------+
| username | ageIs12 |
+----------+---------+
| admin    |       0 |
| guest    |       1 |
| test     |       1 |
+----------+---------+
3 rows in set (0.00 sec)
```

#### JSON\_EXTRACT ####

> 从json文档里抽取数据。如果有参数有NULL或path不存在，则返回NULL。如果抽取出多个path，则返回的数据封闭在一个json array里。

```bash
mysql> SELECT t.*, JSON_EXTRACT(t.extends, '$.open_id') AS open_id
    ->      FROM user t
    ->      WHERE JSON_EXTRACT(t.extends, '$.open_id') = 'yyyyyyyyyyyy'
    -> ;
+----+----------+----------+--------------------------------------------------------------+----------------+
| id | username | password | extends                                                      | open_id        |
+----+----------+----------+--------------------------------------------------------------+----------------+
|  2 | guest    | 123456   | {"age": 12, "open_id": "yyyyyyyyyyyy", "nickname": "访客"}   | "yyyyyyyyyyyy" |
+----+----------+----------+--------------------------------------------------------------+----------------+
1 row in set (0.00 sec)
```

#### JSON\_SEARCH ####

`JSON_SEARCH(json_doc, one_or_all, search_str[, escape_char[, path] ...])`

> 查询包含指定字符串的paths，并作为一个json array返回。如果有参数为NUL或path不存在，则返回NULL。  
> one\_or\_all："one"表示查询到一个即返回；"all"表示查询所有。  
> search\_str：要查询的字符串。 可以用LIKE里的'%'或‘\_’匹配。  
> path：在指定path下查。

```bash
mysql> SET @j = '["abc", [{"k": "10"}, "def"], {"x":"abc"}, {"y":"bcd"}]';
 
mysql> SELECT JSON_SEARCH(@j, 'one', 'abc');
+-------------------------------+
| JSON_SEARCH(@j, 'one', 'abc') |
+-------------------------------+
| "$[0]"                        |
+-------------------------------+
 
mysql> SELECT JSON_SEARCH(@j, 'all', 'abc');
+-------------------------------+
| JSON_SEARCH(@j, 'all', 'abc') |
+-------------------------------+
| ["$[0]", "$[2].x"]            |
+-------------------------------+
 
mysql> SELECT JSON_SEARCH(@j, 'all', 'ghi');
+-------------------------------+
| JSON_SEARCH(@j, 'all', 'ghi') |
+-------------------------------+
| NULL                          |
+-------------------------------+
 
mysql> SELECT JSON_SEARCH(@j, 'all', '10');
+------------------------------+
| JSON_SEARCH(@j, 'all', '10') |
+------------------------------+
| "$[1][0].k"                  |
+------------------------------+
 
mysql> SELECT JSON_SEARCH(@j, 'all', '10', NULL, '$');
+-----------------------------------------+
| JSON_SEARCH(@j, 'all', '10', NULL, '$') |
+-----------------------------------------+
| "$[1][0].k"                             |
+-----------------------------------------+
 
mysql> SELECT JSON_SEARCH(@j, 'all', '10', NULL, '$[*]');
+--------------------------------------------+
| JSON_SEARCH(@j, 'all', '10', NULL, '$[*]') |
+--------------------------------------------+
| "$[1][0].k"                                |
+--------------------------------------------+
 
mysql> SELECT JSON_SEARCH(@j, 'all', '10', NULL, '$**.k');
+---------------------------------------------+
| JSON_SEARCH(@j, 'all', '10', NULL, '$**.k') |
+---------------------------------------------+
| "$[1][0].k"                                 |
+---------------------------------------------+
 
mysql> SELECT JSON_SEARCH(@j, 'all', '10', NULL, '$[*][0].k');
+-------------------------------------------------+
| JSON_SEARCH(@j, 'all', '10', NULL, '$[*][0].k') |
+-------------------------------------------------+
| "$[1][0].k"                                     |
+-------------------------------------------------+
 
mysql> SELECT JSON_SEARCH(@j, 'all', '10', NULL, '$[1]');
+--------------------------------------------+
| JSON_SEARCH(@j, 'all', '10', NULL, '$[1]') |
+--------------------------------------------+
| "$[1][0].k"                                |
+--------------------------------------------+
 
mysql> SELECT JSON_SEARCH(@j, 'all', '10', NULL, '$[1][0]');
+-----------------------------------------------+
| JSON_SEARCH(@j, 'all', '10', NULL, '$[1][0]') |
+-----------------------------------------------+
| "$[1][0].k"                                   |
+-----------------------------------------------+
 
mysql> SELECT JSON_SEARCH(@j, 'all', 'abc', NULL, '$[2]');
+---------------------------------------------+
| JSON_SEARCH(@j, 'all', 'abc', NULL, '$[2]') |
+---------------------------------------------+
| "$[2].x"                                    |
+---------------------------------------------+
 
mysql> SELECT JSON_SEARCH(@j, 'all', '%a%');
+-------------------------------+
| JSON_SEARCH(@j, 'all', '%a%') |
+-------------------------------+
| ["$[0]", "$[2].x"]            |
+-------------------------------+
 
mysql> SELECT JSON_SEARCH(@j, 'all', '%b%');
+-------------------------------+
| JSON_SEARCH(@j, 'all', '%b%') |
+-------------------------------+
| ["$[0]", "$[2].x", "$[3].y"]  |
+-------------------------------+
 
mysql> SELECT JSON_SEARCH(@j, 'all', '%b%', NULL, '$[0]');
+---------------------------------------------+
| JSON_SEARCH(@j, 'all', '%b%', NULL, '$[0]') |
+---------------------------------------------+
| "$[0]"                                      |
+---------------------------------------------+
 
mysql> SELECT JSON_SEARCH(@j, 'all', '%b%', NULL, '$[2]');
+---------------------------------------------+
| JSON_SEARCH(@j, 'all', '%b%', NULL, '$[2]') |
+---------------------------------------------+
| "$[2].x"                                    |
+---------------------------------------------+
 
mysql> SELECT JSON_SEARCH(@j, 'all', '%b%', NULL, '$[1]');
+---------------------------------------------+
| JSON_SEARCH(@j, 'all', '%b%', NULL, '$[1]') |
+---------------------------------------------+
| NULL                                        |
+---------------------------------------------+
 
mysql> SELECT JSON_SEARCH(@j, 'all', '%b%', '', '$[1]');
+-------------------------------------------+
| JSON_SEARCH(@j, 'all', '%b%', '', '$[1]') |
+-------------------------------------------+
| NULL                                      |
+-------------------------------------------+
 
mysql> SELECT JSON_SEARCH(@j, 'all', '%b%', '', '$[3]');
+-------------------------------------------+
| JSON_SEARCH(@j, 'all', '%b%', '', '$[3]') |
+-------------------------------------------+
| "$[3].y"                                  |
+-------------------------------------------+
```

#### JSON\_APPEND/JSON\_ARRAY\_APPEND ####

`JSON_ARRAY_APPEND(json_doc, path, val[, path, val] ...)`

> 在指定path的json array尾部追加val。如果指定path是一个json object，则将其封装成一个json array再追加。如果有参数为NULL，则返回NULL。

```bash
mysql> SET @j = '["a", ["b", "c"], "d"]';
mysql> SELECT JSON_ARRAY_APPEND(@j, '$[1]', 1);
+----------------------------------+
| JSON_ARRAY_APPEND(@j, '$[1]', 1) |
+----------------------------------+
| ["a", ["b", "c", 1], "d"]        |
+----------------------------------+
mysql> SELECT JSON_ARRAY_APPEND(@j, '$[0]', 2);
+----------------------------------+
| JSON_ARRAY_APPEND(@j, '$[0]', 2) |
+----------------------------------+
| [["a", 2], ["b", "c"], "d"]      |
+----------------------------------+
mysql> SELECT JSON_ARRAY_APPEND(@j, '$[1][0]', 3);
+-------------------------------------+
| JSON_ARRAY_APPEND(@j, '$[1][0]', 3) |
+-------------------------------------+
| ["a", [["b", 3], "c"], "d"]         |
+-------------------------------------+
 
mysql> SET @j = '{"a": 1, "b": [2, 3], "c": 4}';
mysql> SELECT JSON_ARRAY_APPEND(@j, '$.b', 'x');
+------------------------------------+
| JSON_ARRAY_APPEND(@j, '$.b', 'x')  |
+------------------------------------+
| {"a": 1, "b": [2, 3, "x"], "c": 4} |
+------------------------------------+
mysql> SELECT JSON_ARRAY_APPEND(@j, '$.c', 'y');
+--------------------------------------+
| JSON_ARRAY_APPEND(@j, '$.c', 'y')    |
+--------------------------------------+
| {"a": 1, "b": [2, 3], "c": [4, "y"]} |
+--------------------------------------+
 
mysql> SET @j = '{"a": 1}';
mysql> SELECT JSON_ARRAY_APPEND(@j, '$', 'z');
+---------------------------------+
| JSON_ARRAY_APPEND(@j, '$', 'z') |
+---------------------------------+
| [{"a": 1}, "z"]                 |
+---------------------------------+
```

#### JSON\_ARRAY\_INSERT ####

`JSON_ARRAY_INSERT(json_doc, path, val[, path, val] ...)`

> 在path指定的json array元素插入val，原位置及以右的元素顺次右移。如果path指定的数据非json array元素，则略过此val；如果指定的元素下标超过json array的长度，则插入尾部。

```bash
mysql> SET @j = '["a", {"b": [1, 2]}, [3, 4]]';
mysql> SELECT JSON_ARRAY_INSERT(@j, '$[1]', 'x');
+------------------------------------+
| JSON_ARRAY_INSERT(@j, '$[1]', 'x') |
+------------------------------------+
| ["a", "x", {"b": [1, 2]}, [3, 4]]  |
+------------------------------------+
mysql> SELECT JSON_ARRAY_INSERT(@j, '$[100]', 'x');
+--------------------------------------+
| JSON_ARRAY_INSERT(@j, '$[100]', 'x') |
+--------------------------------------+
| ["a", {"b": [1, 2]}, [3, 4], "x"]    |
+--------------------------------------+
mysql> SELECT JSON_ARRAY_INSERT(@j, '$[1].b[0]', 'x');
+-----------------------------------------+
| JSON_ARRAY_INSERT(@j, '$[1].b[0]', 'x') |
+-----------------------------------------+
| ["a", {"b": ["x", 1, 2]}, [3, 4]]       |
+-----------------------------------------+
mysql> SELECT JSON_ARRAY_INSERT(@j, '$[2][1]', 'y');
+---------------------------------------+
| JSON_ARRAY_INSERT(@j, '$[2][1]', 'y') |
+---------------------------------------+
| ["a", {"b": [1, 2]}, [3, "y", 4]]     |
+---------------------------------------+
mysql> SELECT JSON_ARRAY_INSERT(@j, '$[0]', 'x', '$[2][1]', 'y');
+----------------------------------------------------+
| JSON_ARRAY_INSERT(@j, '$[0]', 'x', '$[2][1]', 'y') |
+----------------------------------------------------+
| ["x", "a", {"b": [1, 2]}, [3, 4]]                  |
+----------------------------------------------------+
```

#### JSON\_INSERT/JSON\_REPLACE/JSON\_SET ####

`JSON_INSERT(json_doc, path, val[, path, val] ...)`

> 在指定path下插入数据，如果path已存在，则忽略此val（**不存在才插入**）。

```bash
mysql> SET @j = '{ "a": 1, "b": [2, 3]}';
mysql> SELECT JSON_INSERT(@j, '$.a', 10, '$.c', '[true, false]');
+----------------------------------------------------+
| JSON_INSERT(@j, '$.a', 10, '$.c', '[true, false]') |
+----------------------------------------------------+
| {"a": 1, "b": [2, 3], "c": "[true, false]"}        |
+----------------------------------------------------+
```

`JSON_REPLACE(json_doc, path, val[, path, val] ...)`

> 替换指定路径的数据，如果某个路径不存在则略过（**存在才替换**）。如果有参数为NULL，则返回NULL。

```bash
mysql> SET @j = '{ "a": 1, "b": [2, 3]}';
mysql> SELECT JSON_REPLACE(@j, '$.a', 10, '$.c', '[true, false]');
+-----------------------------------------------------+
| JSON_REPLACE(@j, '$.a', 10, '$.c', '[true, false]') |
+-----------------------------------------------------+
| {"a": 10, "b": [2, 3]}                              |
+-----------------------------------------------------+
```

`JSON_SET(json_doc, path, val[, path, val] ...)`

> 设置指定路径的数据（**不管是否存在**）。如果有参数为NULL，则返回NULL。

```bash
mysql> SET @j = '{ "a": 1, "b": [2, 3]}';
mysql> SELECT JSON_SET(@j, '$.a', 10, '$.c', '[true, false]');
+-------------------------------------------------+
| JSON_SET(@j, '$.a', 10, '$.c', '[true, false]') |
+-------------------------------------------------+
| {"a": 10, "b": [2, 3], "c": "[true, false]"}    |
+-------------------------------------------------+
mysql> SELECT JSON_INSERT(@j, '$.a', 10, '$.c', '[true, false]');
+----------------------------------------------------+
| JSON_INSERT(@j, '$.a', 10, '$.c', '[true, false]') |
+----------------------------------------------------+
| {"a": 1, "b": [2, 3], "c": "[true, false]"}        |
+----------------------------------------------------+
mysql> SELECT JSON_REPLACE(@j, '$.a', 10, '$.c', '[true, false]');
+-----------------------------------------------------+
| JSON_REPLACE(@j, '$.a', 10, '$.c', '[true, false]') |
+-----------------------------------------------------+
| {"a": 10, "b": [2, 3]}                              |
+-----------------------------------------------------+
```

#### JSON\_MERGE ####

`JSON_MERGE(json_doc, json_doc[, json_doc] ...)`  
合并多个json文档。规则如下：

 * 如果都是json array，则结果自动merge为一个json array；
 * 如果都是json object，则结果自动merge为一个json object；
 * 如果有多种类型，则将非json array的元素封装成json array再按照规则一进行mege。


```bash
mysql> SELECT JSON_MERGE('[1, 2]', '[true, false]');
+---------------------------------------+
| JSON_MERGE('[1, 2]', '[true, false]') |
+---------------------------------------+
| [1, 2, true, false]                   |
+---------------------------------------+
mysql> SELECT JSON_MERGE('{"name": "x"}', '{"id": 47}');
+-------------------------------------------+
| JSON_MERGE('{"name": "x"}', '{"id": 47}') |
+-------------------------------------------+
| {"id": 47, "name": "x"}                   |
+-------------------------------------------+
mysql> SELECT JSON_MERGE('1', 'true');
+-------------------------+
| JSON_MERGE('1', 'true') |
+-------------------------+
| [1, true]               |
+-------------------------+
mysql> SELECT JSON_MERGE('[1, 2]', '{"id": 47}');
+------------------------------------+
| JSON_MERGE('[1, 2]', '{"id": 47}') |
+------------------------------------+
| [1, 2, {"id": 47}]                 |
+------------------------------------+
```

#### JSON\_REMOVE ####

`JSON_REMOVE(json_doc, path[, path] ...)`

> 移除指定路径的数据，如果某个路径不存在则略过此路径。如果有参数为NULL，则返回NULL。

```bash
mysql> SET @j = '["a", ["b", "c"], "d"]';
mysql> SELECT JSON_REMOVE(@j, '$[1]');
+-------------------------+
| JSON_REMOVE(@j, '$[1]') |
+-------------------------+
| ["a", "d"]              |
+-------------------------+
```

相关函数的例子就列举到这里，还有更多的，就自行去查看文档吧。

### 创建索引 ###

要在JSON列上进行检索，需要对检索的key创建`虚拟列`，然后再虚拟列上创建索引

```sql
ALTER TABLE user
    ADD open_id_virtual VARCHAR(32) 
    GENERATED ALWAYS AS (JSON_EXTRACT(extends, '$.open_id')) VIRTUAL;
```

![2019-08-19-73243][]

```sql
CREATE INDEX open_id_virtual_index ON user(open_id_virtual);
```

![2019-08-19-073245][]

### 其他 ###

MySQL的JSON数据类型已经足够强大了，就是不知道现在的JAVA ORM能否提供完整的支持，这个留到以后再研究。


[2019-08-19-073241]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20190528/ee5e9713-70fe-4885-84d7-5d3e6012c38f.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[2019-08-19-073242]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20190528/50d59a78-e9f6-452f-85bb-63d946fd2422.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[2019-08-19-73243]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20190528/23db6485-f2e4-4ea8-897e-c0ec8fb1480a.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[2019-08-19-073245]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20190528/d4e2f5fe-e7bc-4fb9-b2ff-0c7f5780d06b.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg