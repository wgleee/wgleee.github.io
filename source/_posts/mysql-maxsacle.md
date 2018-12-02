---
title: MaxScale 实现 MySQL 读写分离，负载均衡
date: 2018-04-05 21:43:14
categories: mysql
tags: mysql
---

> maxscale 是 mariadb 公司开发的一套数据库中间件，可以很方便的实现读写分离方案；并且提供了读写分离的负载均衡和高可用性保障。另外 maxscale 对于前端应用而言是透明的，我们可以很方便的将应用迁移到 maxscale 中实现读写分离方案，来分担主库的压力。 maxscale 也提供了sql语句的解析过滤功能。这里我们主要讲解 maxscale 的安装、配置。

<!-- more -->

- 参考文档：[DBAplus 社区](http://dbaplus.cn/news-11-627-1.html)

## 搭建主从集群

> 一主两从

## 安装 MaxScale

- [github 地址](https://github.com/mariadb-corporation/MaxScale)
- [下载地址](https://downloads.mariadb.com/MaxScale)

```bash
yum install https://downloads.mariadb.com/MaxScale/1.4.5/centos/6Server/x86_64/maxscale-1.4.5-1.centos.6.x86_64.rpm
```

## 配置 MaxScale

*在主库创建监控用户，路由用户*

```bash
create user scalemon@'%' identified by "111111";
grant replication slave, replication client on *.* to scalemon@'%';

create user maxscale@'%' identified by "111111";
grant select on mysql.* to maxscale@'%';
```

> 从库会自动同步

*开始配置*

找到 [server1] 部分，修改其中的 address 和 port，指向 master 的 IP 和端口。

复制2次 [server1] 的整块儿内容，改为 [server2] 与 [server3]，同样修改其中的 address 和 port，分别指向 slave1 和 slave2

找到 [MySQL Monitor] 部分，修改 servers 为 server1,server2,server3，修改 user 和 passwd 为之前创建的监控用户的信息（scalemon,111111）

找到 [Read-Write Service] 部分，修改 servers 为 server1,server2,server3，修改 user 和 passwd 为之前创建的路由用户的信息（maxscale,111111）

由于我们使用了 [Read-Write Service]，需要删除或注释另一个服务 [Read-Only Service]，删除其整块儿内容即可。 [Read-Only Listener] 也需要同时删除或注释

```bash
[root@MHA_Maxscale ~]# cat /etc/maxscale.cnf
[maxscale]
threads=1
log_info=1      # 日志级别
logdir=/tmp/    # 指定日志存放路径

[server1]
type=server
address=192.168.79.12
port=3306
protocol=MySQLBackend

[server2]
type=server
address=192.168.79.13
port=3306
protocol=MySQLBackend

[server3]
type=server
address=192.168.79.14
port=3306
protocol=MySQLBackend

[MySQL Monitor]
type=monitor
module=mysqlmon
servers=server1,server2,server3
user=scalemon
passwd=111111
monitor_interval=10000
# 所有 slave 失效后，master 也可正常访问
# 如果为 false slave 全部失效后 master 也将不可用
detect_stale_master=true

[Read-Write Service]
type=service
router=readwritesplit
servers=server1,server2,server3
user=maxscale
passwd=111111
max_slave_connections=100%

[MaxAdmin Service]
type=service
router=cli

[Read-Write Listener]
type=listener
service=Read-Write Service
protocol=MySQLClient
port=4006

[MaxAdmin Listener]
type=listener
service=MaxAdmin Service
protocol=maxscaled
port=6603
```

*启动检查状态*

```bash
[root@MHA_Maxscale ~]# /etc/init.d/maxscale
[root@MHA_Maxscale ~]# netstat -anptl | grep maxscale
tcp        0      0 0.0.0.0:4006                0.0.0.0:*                   LISTEN      1882/maxscale
tcp        0      0 0.0.0.0:6603                0.0.0.0:*                   LISTEN      1882/maxscale
```

- 4006: 是连接 MaxScale 时使用的端口
- 6603: 是 MaxScale 管理器的端口

登录 MaxScale 管理器，查看一下数据库连接状态，默认的用户名和密码是 admin/mariadb

```bash
[root@MHA_Maxscale ~]# maxadmin --user=admin --password=mariadb
MaxScale> list servers
Servers.
-------------------+-----------------+-------+-------------+--------------------
Server             | Address         | Port  | Connections | Status
-------------------+-----------------+-------+-------------+--------------------
server1            | 192.168.79.12   |  3306 |           0 | Master, Running
server2            | 192.168.79.13   |  3306 |           0 | Slave, Running
server3            | 192.168.79.14   |  3306 |           0 | Slave, Running
-------------------+-----------------+-------+-------------+--------------------
```
