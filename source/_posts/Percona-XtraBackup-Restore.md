---
title: Percona-XtraBackup 快速重建 MySQL 从库
date: 2018-03-14 15:56:14
categories: mysql
tags: mysql
---

> 本文将介绍如何使用 xtrabackup 快速重建 mysql 从库

<!-- more -->

## 安装 Percona-XtraBackup

- mysql-5.6 安装 Percona-XtraBackup-2.3.x
- mysql-5.7 安装 Percona-XtraBackup-2.4.x

[下载地址](https://www.percona.com/downloads/XtraBackup/LATEST/)

```bash
yum install http://www.percona.com/downloads/percona-release/redhat/0.1-4/percona-release-0.1-4.noarch.rpm
yum install percona-xtrabackup
```

> Tips: 1. 以上 rpm 安装链接可能会过期，请从官网获取。 2. 如果提示依赖问题请安装 epel 源 `yum install epel-release`


## 备份主库(全备)

```bash
innobackupex --defaults-file=/usr/local/mysql/my.cnf -S /usr/local/mysql/data/mysql.sock --user=root --password=wglee  /data/backup/
```


- --defaults-file: 指定 my.cnf 配置文件
- -S: 指定 mysql.sock 文件路径
- --user: 连接备份数据库使用的用户名
- --password: 密码
- /data/backup/: 备份文件存放的路径


## 创建主库

*配置从库 my.cnf*

```
[mysqld]
server-id=133
```

*拷贝主库备份文件至从库*

```bash
scp -r /data/backup/2018-03-14_15-10-57 slave:/data/backup/
```

*备份还原前期准备*

```bash
# 停止数据库服务
/etc/init.d/mysqld stop
cd /usr/local/mysql
# 将 datadir 重命名，因为 innobackupex 恢复数据时 datadir 不能有数据
mv data data_ori
```

*准备备份*

一般情况下，在备份完成后，数据尚且不能用于恢复操作，因为备份的数据中可能会包含尚未提交的事务或已经提交但尚未同步至数据文件中的事务。因此，此时数据文件仍处理不一致状态。“准备” 的主要作用正是通过回滚未提交的事务及同步已经提交的事务至数据文件也使得数据文件处于一致性状态

```bash
innobackupex --apply-log /data/backup/2018-03-14_15-10-57
```

> 在实现“准备”的过程中，innobackupex 通常还可以使用 --use-memory 选项来指定其可以使用的内存的大小，默认通常为 100M。如果有足够的内存可用，可以多划分一些内存给 prepare 的过程，以提高其完成速度。

*从备份恢复数据至从库*

```bash
innobackupex --defaults-file=/usr/local/mysql/my.cnf --copy-back /data/backup/2018-03-14_15-10-57
```

> 如果服务器剩余空间不足，你可以使用 `--move-back` 替换掉 `--copy-back`

*修改相应权限*

```bash
chown -R mysql.mysql /usr/local/mysql/data
```

*启动从库，配置 slave*

查看备份文件 xtrabackup_binlog_info 获取 MASTER_LOG_FILE, MASTER_LOG_POS。

```bash
CHANGE MASTER TO
MASTER_HOST='<master_host>',
MASTER_USER='<slave_username>',
MASTER_PASSWORD='<slave_password>',
MASTER_LOG_FILE='<see xtrabackup_binlog_info>',
MASTER_LOG_POS=<see xtrabackup_binlog_info>;
```

*启动 slave*

```bash
start slave;
```

*查看从库状态*

```bash
show slave status\G;
```

> 用 innobackupex 备份数据时，–apply-log 处理过的备份数据里有两个文件说明该备份数据对应的 binlog 的文件名和位置。但有时这俩文件说明的位置可能会不同。

> 1. 对于纯 InnoDB 操作，备份出来的数据中上述两个文件的内容是一致的
> 2. 对于 InnoDB 和非事务存储引擎混合操作，xtrabackup_binlog_info 中所示的 position 应该会比 xtrabackup_pos_innodb 所示的数值大。此时应以 xtrabackup_binlog_info 为准；而后者和 apply-log 时 InnoDB recovery log 中显示的内容是一致的，只针对 InnoDB 这部分数据。


> [参考地址](https://segmentfault.com/a/1190000002575399)
