---
title: Mysql
date: 2022-1-23 15:07:16
tags: 学习笔记
categories: 学习笔记
---

## 数据库简单介绍

1、 按照数据库的发展时间顺序，主要出现了以下类型数据库系统：

- 网状型数据库
- 层次型数据库
-  关系型数据库
- 面向对象数据库

上面4中数据库系统中，关系型数据库使用最为广泛。面向对象数据库则是由面向对象语言催生的新型数据库，目前的一些数据库系统，如：SQL Server 2005、Oracle10g等都开始增加面向对象的特性。

 

## 理论知识

### SQL语句基础理论

SQL是操作和检索关系型数据库的标准语言，标准SQL语句可用于操作然后关系型数据库。

标准的SQL语句通常划分为以下类型：

**查询语句：**主要由于select关键字完成，查询语句是SQL语句中最复杂，功能最丰富的语句。

**DML**（Data Munipulation Language，数据操作语言）语句，这组DML语句修改后数据将保持较好的一致性；操作表的语句，如插入、修改、删除等；

**DDL**（Data Definition Language，数据定义语言）语句，操作数据对象的语言，有create、alter、drop。

**DCL**（Data Control Language，数据控制语言）语句，主要有grant、revoke语句。

**事务控制语句：**主要有commit、rollback和savepoint三个关键字完成

DDL语句是操作数据库对象的语句，包括创建create、删除drop、修改alter数据库对象。

### 常见数据库对象

| 对象名称 | 对应关键字 | 描述                                                         |
| -------- | ---------- | ------------------------------------------------------------ |
| 表       | table      | 表是数据库存储的逻辑单元，以行和列的形式存在；列是字段，行就是一条数据记录 |
| 数据字典 |            | 就是系统表，存储数据库相关信息的表，系统表里的数据通常有数据库系统维护。系统表结构和数据，开发人员不应该手动修改，只能查询其中的数据 |
| 视图     | view       | 一个或多个数据表里的数据的逻辑显示，视图就是一张虚拟的表，并不真正存储数据 |
| 约束     | constraint | 执行数据检验规则，用于保证数据完整性的规则                   |
| 索引     | index      | 用于提高查询性能，相当于书的目录                             |
| 函数     | function   | 用于完成一个特定的计算，具有返回值和参数                     |
| 存储过程 | procedure  | 完成某项完整的业务处理，没有返回值，但可通过传出参数将多个值传个调用环境 |
| 触发器   | trigger    | 相当于一个事件的监听器，当数据库发生特定的事件后，触发器被触发，完成响应处理 |

上面的对象都可以通过用create、alter、drop完成相关的创建、修改、删除操作。

### 常用数据类型

| 列类型                                         | 说明                                                         |
| ---------------------------------------------- | ------------------------------------------------------------ |
| tinyint/smallint/mediumint int(integer)/bigint | 1字节、2字节、3字节、4字节、8字节整数，又可分有符号和无符号两种。这些整数类型的区别仅仅表现范围不同 |
| float/double                                   | 单精度、双精度浮点类型                                       |
| decimal(dec)                                   | 精确小数类型，相当于float和double不会产生精度丢失问题        |
| date                                           | 日期类型，不能保存时间。当Java里的Date对象保存到该类型中，时间部分丢失 |
| time                                           | 时间类型，不能保存日期。当Java的Date对象的保存在该类型中，日期部分丢失 |
| datetime                                       | 日期、时间类型                                               |
| timestamp                                      | 时间戳类型                                                   |
| year                                           | 年类型，仅保存年份                                           |
| char                                           | 定长字符串类型                                               |
| varchar                                        | 可变长度字符串类型                                           |
| binary                                         | 定长二进制字符串类型，它以二进制形式保存字符串               |
| varbinary                                      | 可变长度的二进制字符串类型，二进制形式保存字符串             |
| tingblob/blobmediumblob/longblob               | 1字节、2字节、3字节、4字节的二进制大对象，可存存储超图片、音乐等二进制数据，分别可存储：255/64K/16M/4G的大小 |
| tingtext/textmediumtext/longtext               | 1字节、2字节、3字节、4字节的文本对象，可存储超长长度的字符串，分别可存储：255/64K/16M/4G的大小的文本 |
| enum(‘val1’, ‘val2’, …)                        | 枚举类型，该列的值只能是enum括号中出现的值的之一             |
| set(‘value1’, ‘value2’, …)                     | 集合类型，该列的值可以是set中的一个或多个值                  |

##  常用命令

### Mysql连接

```text
mysql -u username -p
```

实例:

```text
mysql -u root -p
```

### 元数据查询

```text
//服务器版本信息
SELECT VERSION( )   
//当前数据库名 (或者返回空)
SELECT DATABASE( )  
//当前用户名
SELECT USER( )  
//服务器状态
SHOW STATUS 
//服务器配置变量
SHOW VARIABLES
```

### 数据库操作

1）创建数据库

语法

```sql
CREATE DATABASE 数据库名;
```

2）查询数据库

语法

```sql
SHOW DATABASES;
```

3）删除数据库

```sql
drop database <数据库名>;
```

4）使用数据库

```sql
use DATABASE;
```

### 创建数据表

语法

```sql
CREATE TABLE table_name (column_name column_type);
```

或者顺便指定索引

```sql
CREATE TABLE table_name (
  column1_name data_type,
  column2_name data_type,
  ...,
  INDEX index_name (column1 [ASC|DESC], column2 [ASC|DESC], ...)
);
```

实例：

```sql
CREATE TABLE IF NOT EXISTS `User`(
   `user_id` INT UNSIGNED AUTO_INCREMENT,
   `user_name` VARCHAR(100) NOT NULL,
   `user_sex` VARCHAR(10) NOT NULL,
   `insert_date` DATE,
   PRIMARY KEY ( `user_id` )
)ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

### 删除数据表

语法

```sql
DROP TABLE table_name ;
```

实例：

```sql
DROP TABLE User;
```

### 修改数据表结构

> 在进行重要的结构修改时，建议先备份数据，并在生产环境中谨慎操作,因为修改时会影响到数据库的性能和运行时间。

1）添加新字段

```sql
ALTER TABLE table_name
ADD column_name data_type;
```

2）修改字段类型

```sql
ALTER TABLE table_name
MODIFY column_name new_data_type;
```

3）修改字段名称

```sql
ALTER TABLE table_name
CHANGE old_column_name new_column_name data_type;
```

4）删除字段

```sql
ALTER TABLE table_name
DROP column_name;
```

5）添加主键约束

```sql
ALTER TABLE table_name
ADD PRIMARY KEY (column_name);
```

6）添加外键约束

```sql
ALTER TABLE table_name
ADD FOREIGN KEY (column_name) REFERENCES referenced_table(ref_column_name);
```

7）添加普通索引

```sql
ALTER TABLE table_name
ADD INDEX index_name (column1 [ASC|DESC], column2 [ASC|DESC], ...);
```

或者

```sql
CREATE INDEX index_name
ON table_name (column1 [ASC|DESC], column2 [ASC|DESC], ...);
```

8）添加唯一索引

```sql
ALTER TABLE table_name
ADD UNIQUE INDEX index_name (column1 [ASC|DESC], column2 [ASC|DESC], ...);
```

或者

```sql
CREATE UNIQUE INDEX index_name
ON table_name (column1 [ASC|DESC], column2 [ASC|DESC], ...);
```

9）删除索引

```sql
ALTER TABLE table_name
DROP INDEX index_name;
```

或者

```sql
DROP INDEX index_name ON table_name;
```

10）重命名表

```sql
ALTER TABLE old_table_name
RENAME TO new_table_name;
```

11）修改表的存储引擎

```sql
ALTER TABLE table_name ENGINE = new_storage_engine;
```

### 复制数据表

如果我们需要完全的复制MySQL的数据表，包括表的结构，索引，默认值等。 如果仅仅使用CREATE TABLE ... SELECT 命令，是无法实现的。下面介绍如何完全复制一张表

> 原始表是User，复制一个新的表叫User_2

1）获取数据表User的建表语句。 命令：SHOW CREATE TABLE User;

2）修改SQL语句的数据表名为User_2，并执行SQL语句。

```sql
CREATE TABLE `user_2` (
  `user_id` int(10) unsigned NOT NULL AUTO_INCREMENT,
  `user_name` varchar(100) NOT NULL,
  `user_sex` varchar(10) NOT NULL,
  `insert_date` date DEFAULT NULL,
  PRIMARY KEY (`user_id`)
) ENGINE=InnoDB AUTO_INCREMENT=3 DEFAULT CHARSET=utf8
```

此时表user_2的结构和原始表user完全一致，但是没有数据。

3）使用INSERT INTO... SELECT 语句拷贝表中的数据

命令：

```sql
INSERT INTO User_2(user_id,user_name,user_sex,insert_date) select user_id,user_name,user_sex,insert_date from user;
```

### 插入数据行

语法

```sql
INSERT INTO table_name ( field1, field2,...fieldN )VALUES( value1, value2,...valueN );
```

实例：

```sql
INSERT INTO User(user_name,user_sex,insert_date) VALUES('admin','男','2023-08-29');
```

### 查询数据行

语法

```sql
SELECT column_name,column_name
FROM table_name
[WHERE Clause]
[LIMIT N][ OFFSET M]
```

实例：

```sql
SELECT user_name,user_sex FROM User WHERE user_name='admin' LIMIT 1;
```

### 更新数据行

语法

```sql
UPDATE table_name SET field1=new-value1, field2=new-value2
[WHERE Clause]
```

实例：

```sql
UPDATE User SET user_sex='女' where user_name='admin';
```

### 删除数据行

语法

```sql
DELETE FROM table_name [WHERE Clause]
```

实例：

```text
DELETE FROM User WHERE user_name='admin';
```

### 模糊查询LIKE

语法

```sql
SELECT field1, field2,...fieldN 
FROM table_name
WHERE field1 LIKE condition1 [AND [OR]] filed2 = 'somevalue'
```

> LIKE 通常与 % 一同使用，类似于一个元字符的搜索

实例：

```sql
SELECT *FROM User WHERE user_name LIKE '%ad%';
```

### 联合查询UNION

语法

```sql
SELECT expression1, expression2, ... expression_n
FROM tables
[WHERE conditions]
UNION [ALL | DISTINCT]
SELECT expression1, expression2, ... expression_n
FROM tables
[WHERE conditions];
```

> UNION ALL: 返回所有结果集，包含重复数据。 UNION: 返回所有结果集，不包含重复数据。

### 排序

语法

```sql
SELECT field1, field2,...fieldN FROM table_name1, table_name2...
ORDER BY field1 [ASC [DESC][默认 ASC]], [field2...] [ASC [DESC][默认 ASC]]
```

实例：

```sql
//按插入时间升序
SELECT * from User ORDER BY insert_date ASC;
```

### 分组

语法

```sql
SELECT column_name, function(column_name)
FROM table_name
WHERE column_name operator value
GROUP BY column_name;
```

实例：

```sql
//按用户名分组
SELECT user_name,count(1) from User GROUP BY user_name
```

### 多表连接查询

- INNER JOIN（内连接,或等值连接）：获取两个表中字段匹配关系的记录。
- LEFT JOIN（左连接）：获取左表所有记录，即使右表没有对应匹配的记录。
- RIGHT JOIN（右连接）： 与 LEFT JOIN 相反，用于获取右表所有记录，即使左表没有对应匹配的记录。

> 我们可以在SELECT, UPDATE 和 DELETE 语句中使用 Mysql 的 JOIN 来联合多表查询。

语法：

```sql
SELECT a.x,b.y from a inner join b on a.id = b.a_id
```

### 正则表达式

> MySQL中使用 REGEXP 操作符来进行正则表达式匹配。

实例：

```sql
//查找user_name字段中以'st'为开头的所有数据：
SELECT user_name FROM User WHERE user_name REGEXP '^ad';
```

### 事务控制

> 在 MySQL 中只有使用了 Innodb 数据库引擎的数据库或表才支持事务。

语法：

```sql
MYSQL 事务处理主要有两种方法：
1、用 BEGIN, ROLLBACK, COMMIT来实现

BEGIN 开始一个事务
ROLLBACK 事务回滚
COMMIT 事务确认
2、直接用 SET 来改变 MySQL 的自动提交模式:

SET AUTOCOMMIT=0 禁止自动提交
SET AUTOCOMMIT=1 开启自动提交
```

### 数据导出

可以使用 SELECT ... INTO OUTFILE 语句导出数据

实例：

```sql
select *from user into outfile '/test.txt';
```

直接执行上述导出会报错：

```sql
ERROR 1290 (HY000): The MySQL server is running with the --secure-file-priv option so it cannot execute this statement
```

看字面意思是和--secure-file-pri这个变量有关系，我们看下这个变量的设置是什么：

```sql
show variables like '%secure%';
```

### 数据导入

1）使用mysql命令导入
语法：

```sql
mysql -u用户名    -p密码    <  要导入的数据库数据(data.sql)
```

以上命令将将备份的整个数据库 data.sql 导入。

2）使用source命令导入

> source 命令导入数据库需要先登录到数库终端

导入步骤：

```sql
mysql> create database gyd;      # 创建数据库
mysql> use gyd;                  # 使用已创建的数据库 
mysql> set names utf8;           # 设置编码
mysql> source /home/gyd/gyd.sql  # 导入备份数据库
```

3）使用 LOAD DATA 导入数据

以下实例中将从当前目录中读取文件 outfile.txt ，将该文件中的数据插入到当前数据库的 user 表中。

```sql
mysql> LOAD DATA LOCAL INFILE 'outfile.txt' INTO TABLE user;
```

LOAD DATA 默认情况下是按照数据文件中列的顺序插入数据的，如果数据文件中的列与插入表中的列不一致，则需要指定列的顺序。

如，在数据文件中的列顺序是 user_sex,user_name, insert_date，但在插入表的列顺序为user_name, user_sex, insert_date，则数据导入语法如下：

```sql
mysql> LOAD DATA LOCAL INFILE 'outfile.txt' 
    -> INTO TABLE user (user_name, user_sex, insert_date);
```

4）使用 mysqlimport 导入数据

```sql
$ mysqlimport -u root -p --local user outfile.txt
password *****
```