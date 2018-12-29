---
title: MySQL GTID 主从同步出错解决方法
date: 2018-12-12 11:06:38
categories: mysql
tags: 
  - mysql
---

> 环境: 两台 mysql 5.6.27 数据库使用 gtid 做了主从同步
> 原因: 由于从库没有限制好权限开发人员在从库插入数据从而导致主从同步失效

<!-- more -->

## 查看主从同步状态

### 主库状态

```
mysql> show master status;
+--------------+----------+--------------+------------------+-------------------------------------------+
| File         | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set                         |
+--------------+----------+--------------+------------------+-------------------------------------------+
| mysql.000002 |     3541 |              | mysql            | 69231693-651b-11e6-a629-000c29b2535c:1-14 |
+--------------+----------+--------------+------------------+-------------------------------------------+
1 row in set (0.00 sec)
```

### 从库状态

```
mysql> show slave status\G
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 192.168.80.10
                  Master_User: repluser
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql.000002
          Read_Master_Log_Pos: 3541
               Relay_Log_File: mysqld-relay-bin.000002
                Relay_Log_Pos: 3743
        Relay_Master_Log_File: mysql.000002
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
              Replicate_Do_DB:
          .........................
  Replicate_Ignore_Server_Ids:
             Master_Server_Id: 10
                  Master_UUID: 69231693-651b-11e6-a629-000c29b2535c
             Master_Info_File: mysql.slave_master_info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State: Slave has read all relay log; waiting for the slave I/O thread to update it
           Master_Retry_Count: 86400
                  Master_Bind:
      Last_IO_Error_Timestamp:
     Last_SQL_Error_Timestamp:
               Master_SSL_Crl:
           Master_SSL_Crlpath:
           Retrieved_Gtid_Set: 69231693-651b-11e6-a629-000c29b2535c:1-14
            Executed_Gtid_Set: 69231693-651b-11e6-a629-000c29b2535c:1-14
                Auto_Position: 1
1 row in set (0.00 sec)
```

> 可以看到主从状态是正常的，注意`Executed_Gtid_Set`参数

### 测试数据同步

```
################ 操作主库 ################
mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| test               |
| tmp                |
+--------------------+
5 rows in set (0.00 sec)

mysql> use tmp;
Database changed

mysql> insert into test values(1,'lily');
Query OK, 1 row affected (0.00 sec)

mysql> select * from test;
+----+------+
| id | name |
+----+------+
|  1 | lily |
+----+------+
1 row in set (0.00 sec)


################ 操作从库 ################
mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| test               |
| tmp                |
+--------------------+
5 rows in set (0.00 sec)

mysql> use tmp;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
mysql> show tables;
+---------------+
| Tables_in_tmp |
+---------------+
| test          |
+---------------+
1 row in set (0.00 sec)

mysql> select * from test;
+----+------+
| id | name |
+----+------+
|  1 | lily |
+----+------+
1 row in set (0.00 sec)
```

> 此时可以看到主从同步是正常的


## 模拟故障

### 在从库的 tmp 库 test 表中增加一条数据

```
# 在插入数据前先看下从库的状态
mysql> show slave status\G
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 192.168.80.10
                  Master_User: repluser
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql.000002
          Read_Master_Log_Pos: 3842
               Relay_Log_File: mysqld-relay-bin.000002
                Relay_Log_Pos: 4044
        Relay_Master_Log_File: mysql.000002
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
              Replicate_Do_DB:
          .....................................
  Replicate_Ignore_Server_Ids:
             Master_Server_Id: 10
                  Master_UUID: 69231693-651b-11e6-a629-000c29b2535c
             Master_Info_File: mysql.slave_master_info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State: Slave has read all relay log; waiting for the slave I/O thread to update it
           Master_Retry_Count: 86400
                  Master_Bind:
      Last_IO_Error_Timestamp:
     Last_SQL_Error_Timestamp:
               Master_SSL_Crl:
           Master_SSL_Crlpath:
           Retrieved_Gtid_Set: 69231693-651b-11e6-a629-000c29b2535c:1-15
            Executed_Gtid_Set: 69231693-651b-11e6-a629-000c29b2535c:1-15
                Auto_Position: 1
1 row in set (0.00 sec)

# 增加一条数据
mysql> insert into test values(2,'tom');
Query OK, 1 row affected (0.02 sec)
# 在查看从库状态
mysql> show slave status\G
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 192.168.80.10
                  Master_User: repluser
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql.000002
          Read_Master_Log_Pos: 3842
               Relay_Log_File: mysqld-relay-bin.000002
                Relay_Log_Pos: 4044
        Relay_Master_Log_File: mysql.000002
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
              Replicate_Do_DB:
          .........................................
  Replicate_Ignore_Server_Ids:
             Master_Server_Id: 10
                  Master_UUID: 69231693-651b-11e6-a629-000c29b2535c
             Master_Info_File: mysql.slave_master_info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State: Slave has read all relay log; waiting for the slave I/O thread to update it
           Master_Retry_Count: 86400
                  Master_Bind:
      Last_IO_Error_Timestamp:
     Last_SQL_Error_Timestamp:
               Master_SSL_Crl:
           Master_SSL_Crlpath:
           Retrieved_Gtid_Set: 69231693-651b-11e6-a629-000c29b2535c:1-15
            Executed_Gtid_Set: 29d0bc3d-65a7-11e6-a9b8-000c29c171cb:1,
69231693-651b-11e6-a629-000c29b2535c:1-15
                Auto_Position: 1
1 row in set (0.00 sec)

```

> 可以发现从库的`Executed_Gtid_Set:`由一条变成两条了

### 在次测试主从同步

```
# 在主库 tmp 库的 test 表增加一条数据
mysql> insert into test values(2,'jom');
Query OK, 1 row affected (0.01 sec)

mysql> select * from test;
+----+------+
| id | name |
+----+------+
|  1 | lily |
|  2 | jom  |
+----+------+
2 rows in set (0.00 sec)

# 查看从库的表
mysql> select * from  test;
+----+------+
| id | name |
+----+------+
|  1 | lily |
|  2 | tom  |
+----+------+
2 rows in set (0.00 sec)
# 此时从库表中的数据没有改变
# 在次查看从库状态
mysql> show slave status\G
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 192.168.80.10
                  Master_User: repluser
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql.000002
          Read_Master_Log_Pos: 4141
               Relay_Log_File: mysqld-relay-bin.000002
                Relay_Log_Pos: 4044
        Relay_Master_Log_File: mysql.000002
             Slave_IO_Running: Yes
            Slave_SQL_Running: No
              Replicate_Do_DB:
          Replicate_Ignore_DB: mysql
           Replicate_Do_Table:
       Replicate_Ignore_Table:
      Replicate_Wild_Do_Table:
  Replicate_Wild_Ignore_Table:
                   Last_Errno: 1062
                   Last_Error: Worker 1 failed executing transaction '69231693-651b-11e6-a629-000c29b2535c:16' at master log mysql.000002, end_log_pos 4110; Could notexecute Write_rows event on table tmp.test; Duplicate entry '2' for key 'PRIMARY', Error_code: 1062; handler error HA_ERR_FOUND_DUPP_KEY; the event's master log mysql.000002, end_log_pos 4110
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 3842
              Relay_Log_Space: 4548
              Until_Condition: None
               Until_Log_File:
                Until_Log_Pos: 0
           Master_SSL_Allowed: No
           Master_SSL_CA_File:
           Master_SSL_CA_Path:
              Master_SSL_Cert:
            Master_SSL_Cipher:
               Master_SSL_Key:
        Seconds_Behind_Master: NULL
Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error:
               Last_SQL_Errno: 1062
               Last_SQL_Error: Worker 1 failed executing transaction '69231693-651b-11e6-a629-000c29b2535c:16' at master log mysql.000002, end_log_pos 4110; Could notexecute Write_rows event on table tmp.test; Duplicate entry '2' for key 'PRIMARY', Error_code: 1062; handler error HA_ERR_FOUND_DUPP_KEY; the event's master log mysql.000002, end_log_pos 4110
  Replicate_Ignore_Server_Ids:
             Master_Server_Id: 10
                  Master_UUID: 69231693-651b-11e6-a629-000c29b2535c
             Master_Info_File: mysql.slave_master_info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State:
           Master_Retry_Count: 86400
                  Master_Bind:
      Last_IO_Error_Timestamp:
     Last_SQL_Error_Timestamp: 160819 08:59:56
               Master_SSL_Crl:
           Master_SSL_Crlpath:
           Retrieved_Gtid_Set: 69231693-651b-11e6-a629-000c29b2535c:1-16
            Executed_Gtid_Set: 29d0bc3d-65a7-11e6-a9b8-000c29c171cb:1,
69231693-651b-11e6-a629-000c29b2535c:1-15
                Auto_Position: 1
1 row in set (0.00 sec)

```

> 从上面的信息就可以看到从库报错了


### 开始解决

从上面可以看到从库同步出现了故障，因为数据库的表中有一个id列，在生产环境中id列一般为主键，不能重复，而且可能还是自增长的，现在主库与从库两个id冲忽，所以才报错的。

#### 方法1

##### 直接跳过错误的GTID事务

```
mysql> stop slave;
Query OK, 0 rows affected (0.02 sec)

mysql> reset slave;
Query OK, 0 rows affected (0.01 sec)

mysql> set global gtid_purged="69231693-651b-11e6-a629-000c29b2535c:1-16";
Query OK, 0 rows affected (0.02 sec)

mysql> change master to
    -> master_host='192.168.80.10',
    -> master_user='repluser',
    -> master_password='123456',
    -> master_auto_position=1;
Query OK, 0 rows affected, 2 warnings (0.04 sec)

mysql> start slave;
Query OK, 0 rows affected, 1 warning (0.03 sec)

mysql> show slave status \G
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 192.168.80.10
                  Master_User: repluser
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql.000002
          Read_Master_Log_Pos: 4141
               Relay_Log_File: mysqld-relay-bin.000002
                Relay_Log_Pos: 396
        Relay_Master_Log_File: mysql.000002
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
              Replicate_Do_DB:
          Replicate_Ignore_DB: mysql
           Replicate_Do_Table:
       Replicate_Ignore_Table:
      Replicate_Wild_Do_Table:
  Replicate_Wild_Ignore_Table:
                   Last_Errno: 0
                   Last_Error:
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 4141
              Relay_Log_Space: 601
              Until_Condition: None
               Until_Log_File:
                Until_Log_Pos: 0
           Master_SSL_Allowed: No
           Master_SSL_CA_File:
           Master_SSL_CA_Path:
              Master_SSL_Cert:
            Master_SSL_Cipher:
               Master_SSL_Key:
        Seconds_Behind_Master: 0
Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error:
               Last_SQL_Errno: 0
               Last_SQL_Error:
  Replicate_Ignore_Server_Ids:
             Master_Server_Id: 10
                  Master_UUID: 69231693-651b-11e6-a629-000c29b2535c
             Master_Info_File: mysql.slave_master_info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State: Slave has read all relay log; waiting for the slave I/O thread to update it
           Master_Retry_Count: 86400
                  Master_Bind:
      Last_IO_Error_Timestamp:
     Last_SQL_Error_Timestamp:
               Master_SSL_Crl:
           Master_SSL_Crlpath:
           Retrieved_Gtid_Set:
            Executed_Gtid_Set: 69231693-651b-11e6-a629-000c29b2535c:1-16
                Auto_Position: 1
1 row in set (0.00 sec)
```

##### 测试主从同步

```
# 主库
mysql> insert into test values(3,'jom');
Query OK, 1 row affected (0.01 sec)

mysql> select * from test;
+----+------+
| id | name |
+----+------+
|  1 | lily |
|  2 | jom  |
|  3 | jom  |
+----+------+
3 rows in set (0.00 sec)
# 从库
mysql> select * from test;
+----+------+
| id | name |
+----+------+
|  1 | lily |
|  2 | tom  |
|  3 | jom  |
+----+------+
3 rows in set (0.00 sec)
```

> 同步恢复正常，此方法有一个问题就是会导致主从的数据不一致
> 最好就是先恢复数据库正常运行，然后在找时间重建从库
