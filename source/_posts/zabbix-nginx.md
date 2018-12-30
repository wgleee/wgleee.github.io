---
title: ZABBIX 监控 NGINX
date: 2018-12-18 11:40:31
categories: zabbix
tags:
  - zabbix
---

本文将介绍如何利用 Zabbix 监控 NGINX 各项性能指标及实时监控各站点的 HTTP 状态码

<!-- more -->

## 配置 nginx

```bash
local> cd /usr/local/nginx/vhost
local> cat > nginx_status.conf
server {
    listen 80 default;
    server_name localhost;
    index index.html index.htm;
    access_log off;

    location / {
        return 444;
    }

    location /nginx_status {
        stub_status on;
        auth_basic "nginx status";
        allow 127.0.0.1;
        deny all;
    }
}
```

{% note info %}
nginx 编译时需要添加 `--with-http_stub_status_module` 模块
{% endnote %}

## 编写 zabbix 监控脚本

**nginx 状态监控脚本: `nginx_status.sh`**

```bash
#!/bin/bash
NGINX_STATUS='http://127.0.0.1/nginx_status'

function ping {
    /sbin/pidof nginx &>/dev/null && echo 1 || echo 0
}
function active {
    /usr/bin/curl -s "$NGINX_STATUS" | grep 'Active' | awk '{print $NF}'
}
function reading {
    /usr/bin/curl -s "$NGINX_STATUS" | grep 'Reading' | awk '{print $2}'
}
function writing {
    /usr/bin/curl -s "$NGINX_STATUS" | grep 'Writing' | awk '{print $4}'
}
function waiting {
    /usr/bin/curl -s "$NGINX_STATUS" | grep 'Waiting' | awk '{print $6}'
}
function accepts {
    /usr/bin/curl -s "$NGINX_STATUS" | awk NR==3 | awk '{print $1}'
}
function handled {
    /usr/bin/curl -s "$NGINX_STATUS" | awk NR==3 | awk '{print $2}'
}
function requests {
    /usr/bin/curl -s "$NGINX_STATUS" | awk NR==3 | awk '{print $3}'
}

$1
```

**nginx 日志自动发现脚本: `discovery_logfiles.py`**

```python
#!/bin/python

from __future__ import print_function

import os
import json

config_path = '/usr/local/nginx/conf/vhost'
nginx_path = '/usr/local/nginx/'
ignore = ['nginx_status.conf']
logfiles = {"data":[]}

configs = [os.path.join(config_path, f) for f in os.listdir(config_path)
            if f not in ignore]

def get_logfile(confile):
    with open(confile) as f:
        for line in f:
            if 'access_log' not in line:
                continue
            if len(line.strip(';').split()) == 2:
                continue
            logfile = line.strip(';').split()[1]
    if logfile.startswith('logs'):
        logfile = os.path.join(nginx_path, logfile)
    if os.path.isfile(logfile):
        appname = os.path.basename(confile).split('.')[0]
        return logfiles['data'].append({"{#LOGFILE}": logfile, "{#APPNAME}": appname})

for config in configs:
    get_logfile(config)

print(json.dumps(logfiles))
```

**上传监控脚本至: /usr/local/zabbix/scripts**

```bash
local> mkdir /usr/local/zabbix/scripts
local> cd /usr/local/zabbix/scripts
# 上传监控脚本到此目录下并添加执行权限 `chmod +x`
```

## 配置 zabbix-agent

**修改 `zabbix_agentd.conf` 配置文件，添加以下配置**

```
Include=/usr/local/zabbix/etc/zabbix_agentd.conf.d/*.conf
```

**添加自定义监控项**

```bash
local> cd /usr/local/zabbix/etc/zabbix_agentd.conf.d
local> cat nginx_userparameter.conf
UserParameter=nginx.status[*],/usr/local/zabbix/scripts/nginx_status.sh $1
UserParameter=nginx.discovery,/usr/bin/python /usr/local/zabbix/scripts/discovery_logfiles.py
UserParameter=nginx.status_code[*],/bin/grep "$(date +'\[%d/%b/%Y:%H:%M:')" "$1" | /bin/grep -o "HTTP/[1-2].[0-1]\" $2" | wc -l
```

**重启 zabbix-agten**

```
/etc/init.d/zabbix_agentd restart
```

## 添加监控模板

**创建 nginx 监控模板**

    1. 创建监控模板名称为: `Template App NGINX` 群组为: `Templates/Applications`
    2. 创建应用集名称为: nginx

{% asset_img zabbix-1.jpg %}
{% asset_img zabbix-1.1.jpg %}

**创建自动发现规则**

    1. 点击自动发现规则
    2. 创建自动发现规则
    3. 键值输入: `nginx.discovery`
    4. 间隔为1小时
    5. 配置过滤器 (如图所示)

{% asset_img zabbix-2.jpg %}
{% asset_img zabbix-3.jpg %}

**创建 nginx http 状态码监控项：添加监控原型**

    1. 监控原型
    2. 创建监控项原型
    3. 例如: 创建监控 `503` 状态码监控项, 其他状态码监控配置只需将 `503` 改为相应的状态码即可

{% asset_img zabbix-4.jpg %}

**创建 nginx 状态监控项**

    1. 打开 nginx 监控模板
    2. 点击监控项
    3. 创建监控项

{% asset_img zabbix-5.jpg %}
{% asset_img zabbix-6.jpg %}

