---
title: 使用 shadowscoks 配置 socks5 代理
date: 2018-07-24 19:58:15
categories: python
tags: shadowsocks
---

> [Shadowsocks](https://shadowsocks.org) 是一个轻量级且安全的 Socks5 代理(有多种语言的版本)。如果你需要访问国外资源同时又拥有国外的VPS，那么使用 [Shadowsocks](https://shadowsocks.org) 搭建一个翻{防屏蔽}墙服务器是一件很轻松的事情！

<!-- more -->

## 安装 shadowsocks

```bash
pip install shadowsocks
# 安装完成后，shadowsocks 提供以下两个命令供使用
/usr/local/bin/sslocal
/usr/local/bin/ssserver
```

> 请确保您的 Python 版本为 2.6 或 2.7。

## 配置 shadowsocks

Shadowsocks 支持 JSON 格式的配置文件(config.json)，同时也支持命令行模式，下面分别介绍服务端与客户端的配置

### 服务端

```json
{
    "server":"my_server_ip",
    "server_port":8388,
    "password":"barfoo!",
    "timeout":600,
    "method":"chacha20-ietf-poly1305"
}
```

*字段说明:*

- server: 您服务器的域名或者 IP (IPv4/IPv6).
- server_port: shadowsocks 服务监听端口.
- password: 验证密码.
- timeout: 连接超时时间，单位为秒.
- method: 验证加密码方法.

*启动服务*

```bash
ssserver -c config.json -d start
```

> Shadowsocks 启动后可以通过 `netstat -anptl | grep python` 查看端口监控状态

### 客户端

```json
{
    "server":"my_server_ip",
    "server_port":8388,
    "local_port":1080,
    "password":"barfoo!",
    "timeout":600,
    "method":"chacha20-ietf-poly1305"
}
```

*字段说明:*

- server: 您服务器的域名或者 IP (IPv4/IPv6).
- server_port: shadowsocks 服务监听端口.
- local_port: shadowscoks 客户端监听端口.
- password: 验证密码(与服务端一致).
- timeout: 连接超时时间，单位为秒.
- method: 验证加密码方法(与服务端一致).

*启动服务*

```bash
sslocal -c config.json -d start
```

{% note primary %}
Shadowsocks 客户端启动并连接服务端成功后就可以正常进行 sock 网络代理了。
操作方法：
1. 浏览器插件： `Google-chrome` 浏览器可以使用 `SwitchyOmega` 插件进行代理操作。
2. 系统代理： 配置系统代理即可。

> 代理地址为 `127.0.0.1:1080`, 当然您的端口得看您自己的配置文件的配置了。
{% endnote %}
