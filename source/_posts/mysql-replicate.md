---
title: MySQL 传统主从同步配置
date: 2018-04-03 15:59:14
categories: mysql
tags: 
  - mysql
---

> 生产环境中为扩展提高 mysql 的性能，往往需要配置主从同步，mysql 主从同步方式有基于传统和新的 GTID 模式两种，本文将介绍传统的主从同步方式.

<!-- more -->

*环境说明*

```bash
[root@master ~]# cat /etc/redhat-release
CentOS release 6.8 (Final)
[root@master ~]# uname -r
2.6.32-642.el6.x86_64
[root@master ~]# uname -m
x86_64
[root@db_master ~]# mysql --version
mysql  Ver 14.14 Distrib 5.6.39, for Linux (x86_64) using  EditLine wrapper
```

*服务器列表*

- master: 192.168.79.12/24
- slave1: 192.168.79.13/24
- slave2: 192.168.79.14/24

> MySQL安装请参考：[MySQL5.6源码编译安装]({% post_url 2018-04-03-mysql5.6-source-installation %})

## master

**启用 binlog**

```bash
[root@master ~]# cat /usr/local/mysql/etc/my.cnf
[mysqld]

# BINARY LOGGING #
server-id                      = 12
log-bin                        = /usr/local/mysql/data/mysql-bin
binlog-format                  = row
expire-logs-days               = 7
sync-binlog                    = 1
```

**创建同步账号**

```bash
mysql> grant replication slave on *.* to 'rsync'@'192.168.79.%' identified by '123456';
mysql> flush privileges;
```

**导出数据**
用于创建 *slave*

```bash
mysqldump -uroot -p -A --events -B -x --master-data=1 | gzip > all.sql.gz
```

- -A, –all-databases: 备份所有数据库
- -E, –events: 备份事件
- -B, –databases: 备份的数据库
- -x, –lock-all-tables：锁定所有数据库的所有表
- –master-data=1: 等于1时会将 CHANGE MASTER 命令加入备份文件中


## slave

*my.cnf*

从库(slave1)用于备份可以启用 binlog, 如果用于读操作可以不启用,只配置 server-id 即可.

```bash
[root@slave1 ~]# cat /usr/local/mysql/etc/my.cnf
[mysqld]
server-id                      = 13
log-bin                        = /usr/local/mysql/data/mysql-bin
binlog-format                  = row
expire-logs-days               = 7
sync-binlog                    = 1
log-slave-updates
```

**从 master 恢复数据**

```bash
[root@slave1 ~]# gzip -d all.sql.gz
[root@slave1 ~]# mysql -uroot -p < all.sql
```

**设置主从同步**

```bash
# 从备份文件找到 CHANGE MASTER 命令
[root@slave1 ~]# more all.sql
-- CHANGE MASTER TO MASTER_LOG_FILE='mysql-bin.000002', MASTER_LOG_POS=1307;
# 配置 slave
mysql> CHANGE MASTER TO
    -> MASTER_HOST='192.168.79.12',
    -> MASTER_PORT=3306,
    -> MASTER_USER='rsync',
    -> MASTER_PASSWORD='123456',
    -> MASTER_LOG_FILE='mysql-bin.000002',
    -> MASTER_LOG_POS=1307;
mysql> start slave;
mysql> show slave status\G
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 192.168.79.12
                  Master_User: rsync
                  Master_Port: 3306
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
        Seconds_Behind_Master: 0
...
```

> `slave2` 和 `slave1` 操作相同，重复即可. 配置完成可以进行主从同步测试
