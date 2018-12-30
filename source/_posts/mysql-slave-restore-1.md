---
title: 快速重建 MYSQL 从库 (mysqldump)
date: 2018-03-14 10:28:14
categories: mysql
tags:
  - mysql
---

工作中使用 MYSQL 5.6 传统同步方式，从库难免会出现各种问题需要恢复或者重建，本文将介绍如何快速创建/重建从库

<!-- more -->

## 全备主库

通过以下命令全备主库

```bash
mysqldump -uroot -p'wglee.org' -A --events -B -x --master-data=1 | gzip > $(date +%F).sql.gz
```

- -A, --all-databases: 备份所有数据库
- -E, --events: 备份事件
- -B, --databases:  备份的数据库
- -x, --lock-all-tables：锁定所有数据库的所有表
- --master-data=1: 等于1时会将 CHANGE MASTER 命令加入备份文件中

{% note info %}
注意，备份过程中会锁表，请不要在业务运行过程中执行，切记... 将备份文件复制至从库
{% endnote %}

## 创建从库

### 配置从库的 my.cnf (略)
### 查找到备份文件中的 `CHANGE MASTER` 语句, 获取 `MASTER_LOG_FILE`, `MASTER_LOG_POS` 配置

```
grep '^CHANGE MASTER' 2018-03-14.sql
```

### 将备份文件导入至从库
### 在从库上执行 `CHANGE MASTER` 语句, 启用主从同步功能

```bash
CHANGE MASTER TO
MASTER_USER='sync',
MASTER_PORT=3306,
MASTER_HOST='192.168.79.132',
MASTER_PASSWORD='123456',
MASTER_LOG_FILE='mysql-bin.000003',
MASTER_LOG_POS=120;
```

### 检查从库状态

```bash
show slave status\G
```

{% note info %}
最后测试主从同步
{% endnote %}



