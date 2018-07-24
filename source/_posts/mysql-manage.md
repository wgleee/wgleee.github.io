---
title: MySQL 管理
date: 2018-01-14 15:56:14
categories: mysql
tags:
    - mysql
---

> MySQL 数据库基本管理命令，创建用户，授权，设置密码，备份，恢复，表结构更改等...

<!-- more -->

##  用户权限管理

> 进行数据库基本信息相关更改后请使用 `flush privileges;` 刷新数据库信息

### 创建用户

```bash
# 创建用户（默认密码为空）
mysql> create user 'username'@'host';

# 创建用户并设置密码
mysql> create user 'username'@'host' identified by 'password';
```

### 删除用户

```bash
mysql> drop user 'username'@'host';
```

### 更改密码

```bash
# 更改密码 （只对当前登录账号有效）
mysql> set password=password('123456');

# 2. 更改指定用户的密码
mysql> set password for 'username'@'host'=password('123456');
```

### 查询用户权限

```bash
# 查询当前账号的权限
mysql> show grants;

# 查询指定账号的权限
mysql> show grants for 'user'@'host';
```

### 用户授权

```bash
# 对用户授权（如果用户存在就增加权限，不存在就创建用户不过密码为空）
mysql> grant privileges on databasename.tablename to 'username'@'host';

# 对用户授权并设置密码（如果用户存在就增加权限，不存在就创建用户）
mysql> grant privileges on databasename.tablename to 'username'@'host' identified by 'password';
```

> privileges: 权限列表以逗号隔开，例如： select, insert,update

### 用户权限回收

```bash
mysql> revoke privilege on databasename.tablename from 'user'@'host';
```

> 注：数据库名要用反撇号引起，或者不用


## 数据库

### 数据库的基本操作

```bash
# 显示数据库
mysql> show databases;

# 创建数据库
mysql> create database DATABASENAME;

# 查看数据库创建语句
mysql> show create database DATABASENAME

# 删除数据库
mysql> drop database DATABASENAME;
```

### 备份数据库数据及表结构

```bash
# 备份整个数据库
[root@localhost ~]# mysqldump -uroot -p -A > all.sql

# 备份整个数据库的结构
[root@localhost ~]# mysqldump -uroot -p -A -d > all.sql

# 备份单个数据库
[root@localhost ~]# mysqldump -uroot -p DATABASENAME > DATABASENAME.sql

# 一次备份多个数据库
[root@localhost ~]# mysqldump -uroot -p --databases db1 db2 > dbs.sql

# 备份数据库中指定的表
[root@localhost ~]# mysqldump -uroot -p DATABASENAME TABLENAME > DATABASENAME_TABLENAME.sql

# 一次备份数据库中指定的多张表
[root@localhost ~]# mysqldump -uroot -p DATABASENAME t1 t2 > DATABASENAME_ts.sql
```

- `-A`, `--all-databases` : 备份所有数据库
- `-d`, `--no-data` ：只导出表结构

### 导出函数或者存储过程


```bash
mysqldump -hHOSTNAME -uUSERNAME -pPASSWORD -ntd -R DATABASENAME > DATABASENAME.sql
```

- `-ntd` 是表示导出存储过程；
- `-R` 是表示导出函数

### 恢复数据库数据

```bash
# 使用系统命令
[root@localhost ~]# mysql -uroot DATABASENAME < DATABASENAME.sql

# 使用 source 命令
mysql> use lwg;
Database changed
mysql> source /root/lwg.sql;
```

> *Tips*: 恢复数据时，如果数据库不存在需要先创建


## 数据表

### 表的基本操作

```bash
# 查看数据库下所有的表
mysql> show tables;

# 创建表
mysql> CREATE TABLE `TABLENAME` (
  `id` int(10) NOT NULL PRIMARY KEY AUTO_INCREMENT,
  `user` varchar(30) NOT NULL,
  `password` varchar(30) NOT NULL
) ENGINE=MyISAM DEFAULT CHARSET=utf8;

# 显示表结构
mysql> desc TABLENAME;

# 显示表创建语句
mysql> show create table TABLENAME;

# 清空表数据
mysql> truncate table TABLENAME;
mysql> delete from TABLENAME;
```

> 不带 `where` 参数的 `delete` 语句可以删除 `mysql` 表中所有内容
> 使用 `truncate table` 也可以清空 `mysql` 表中所有内容。
> 效率上 `truncate` 比 `delete` 快，但 `truncate` 删除后不记录 `mysql` 日志，不可以恢复数据。
> `delete` 的效果有点像将 `mysql` 表中所有记录一条一条删除到删完，
> 而 `truncate` 相当于保留 `mysql` 表的结构，重新创建了这个表，所有的状态都相当于新表。
> 所以 `delete` 不会重置 ID 列，而 `truncat` 会重置。

### 表 `alter` 的相关操作

```bash
# 增加一个字段(一列),并放到第一列的位置 (first)
mysql> desc users;
+------------+----------+------+-----+---------+-------+
| Field      | Type     | Null | Key | Default | Extra |
+------------+----------+------+-----+---------+-------+
| username   | char(30) | NO   | PRI | NULL    |       |
| userpasswd | char(20) | NO   |     | 123456  |       |
+------------+----------+------+-----+---------+-------+
2 rows in set (0.00 sec)

mysql> alter table users add column id int not null first;
Query OK, 0 rows affected (0.08 sec)
Records: 0  Duplicates: 0  Warnings: 0

mysql> desc users;
+------------+----------+------+-----+---------+-------+
| Field      | Type     | Null | Key | Default | Extra |
+------------+----------+------+-----+---------+-------+
| id         | int(11)  | NO   |     | NULL    |       |
| username   | char(30) | NO   | PRI | NULL    |       |
| userpasswd | char(20) | NO   |     | 123456  |       |
+------------+----------+------+-----+---------+-------+
3 rows in set (0.00 sec)

# 删除一个字段
mysql> alter table users drop userpasswd;
Query OK, 0 rows affected (0.05 sec)
Records: 0  Duplicates: 0  Warnings: 0

mysql> desc users;
+----------+----------+------+-----+---------+-------+
| Field    | Type     | Null | Key | Default | Extra |
+----------+----------+------+-----+---------+-------+
| id       | int(11)  | NO   |     | NULL    |       |
| username | char(30) | NO   | PRI | NULL    |       |
+----------+----------+------+-----+---------+-------+
2 rows in set (0.00 sec)

# 更改列的字段类型
mysql> alter table users modify username varchar(100);
Query OK, 2 rows affected (0.14 sec)
Records: 2  Duplicates: 0  Warnings: 0

mysql> desc users;
+----------+--------------+------+-----+---------+-------+
| Field    | Type         | Null | Key | Default | Extra |
+----------+--------------+------+-----+---------+-------+
| id       | int(11)      | NO   |     | NULL    |       |
| username | varchar(100) | NO   | PRI |         |       |
+----------+--------------+------+-----+---------+-------+
2 rows in set (0.00 sec)

# 更改列名及字段类型
mysql> alter table users change username user varchar(20);
Query OK, 2 rows affected (0.03 sec)
Records: 2  Duplicates: 0  Warnings: 0

mysql> desc users;
+-------+-------------+------+-----+---------+-------+
| Field | Type        | Null | Key | Default | Extra |
+-------+-------------+------+-----+---------+-------+
| id    | int(11)     | NO   |     | NULL    |       |
| user  | varchar(20) | NO   | PRI |         |       |
+-------+-------------+------+-----+---------+-------+
2 rows in set (0.00 sec)

# 修改表的存储引擎
mysql> show create table users;
+-------+---------------------------------------+
| Table | Create Table                          |
+-------+---------------------------------------+
| users | CREATE TABLE `users` (
  `id` int(11) NOT NULL,
  `user` varchar(20) NOT NULL DEFAULT '',
  PRIMARY KEY (`user`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 |
+-------+---------------------------------------+
1 row in set (0.00 sec)

mysql> alter table users ENGINE=myisam;
Query OK, 2 rows affected (0.01 sec)
Records: 2  Duplicates: 0  Warnings: 0

# 这里我们使用另一种方法查询表的默认引擎
mysql> show table status from lwg where name='users'\G
*************************** 1. row ***************************
           Name: users
         Engine: MyISAM
        Version: 10
     Row_format: Dynamic
           Rows: 2
 Avg_row_length: 20
    Data_length: 40
Max_data_length: 281474976710655
   Index_length: 2048
      Data_free: 0
 Auto_increment: NULL
    Create_time: 2017-08-25 04:15:46
    Update_time: 2017-08-25 04:15:46
     Check_time: NULL
      Collation: utf8_general_ci
       Checksum: NULL
 Create_options:
        Comment:
1 row in set (0.00 sec)

```
