---
title: CentOS 7.x 配置 bond
date: 2018-12-31 10:24:56
categories: linux
tags:
  - 双网卡绑定
---

*实验环境*

```bash
[root@localhost ~]# cat /etc/redhat-release
CentOSLinux release 7.2.1511(Core)
[root@localhost ~]# uname -r
3.10.0-327.el7.x86_64
```

*实验要求*

linux 服务器 `eno33554960` 与 `eno50332184` 两张网卡配置 `bond` (如果要配置多个按照这个流程重复操作即可)

<!-- more -->


## 配置 bond

### 查看网卡信息

```bash
[root@localhost ~]# ip addr
1: lo:<LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN
link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
inet 127.0.0.1/8 scope host lo
valid_lft forever preferred_lft forever
inet6 ::1/128 scope host
valid_lft forever preferred_lft forever
2: eno16777736:<BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
link/ether 00:0c:29:07:2c:86 brd ff:ff:ff:ff:ff:ff
inet 192.168.92.11/24 brd 192.168.92.255 scope global eno16777736
valid_lft forever preferred_lft forever
inet6 fe80::20c:29ff:fe07:2c86/64 scope link
valid_lft forever preferred_lft forever
3: eno33554960:<BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
link/ether 00:0c:29:07:2c:90 brd ff:ff:ff:ff:ff:ff
4: eno50332184:<BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
link/ether 00:0c:29:07:2c:9a brd ff:ff:ff:ff:ff:ff
```

### 备份网卡配置文件

```bash
[root@localhost ~]# cd /etc/sysconfig/network-scripts/
[root@localhost network-scripts]# mkdir /tmp/net_bak
[root@localhost network-scripts]# cp ifcfg-*/tmp/net_bak/
[root@localhost network-scripts]# ls /tmp/net_bak/
ifcfg-eno16777736 ifcfg-eno33554960 ifcfg-eno50332184 ifcfg-eno67109408 ifcfg-eno83886632 ifcfg-lo
```

### 使用`nmcli`命令配置`bond`

```bash
# 生成bond配置文件
[root@localhost network-scripts]# nmcli connection add type bond ifname bond0 mode 1
# 将网卡`eno33554960`与`eno50332184`绑定到bond0
[root@localhost network-scripts]# nmcli connection add type bond-slave ifname eno33554960 master bond0
[root@localhost network-scripts]# nmcli connection add type bond-slave ifname eno50332184 master bond0
# 查看生成的配置文件
[root@localhost network-scripts]# ls ifcfg-bond-*
ifcfg-bond-bond0 ifcfg-bond-slave-eno33554960 ifcfg-bond-slave-eno50332184
```

**bond 的 mode 如下：**

```
balance-rr      (0) 轮询模式，负载均衡（bond默认的模式）
active-backup 	(1) 主备模式（常用）
balance-xor 	(2)
broadcast       (3）
802.3ad         (4) 聚合模式
balance-tlb 	(5)
balance-alb 	(6)
```

### 修改bond0网卡配置

```bash
[root@localhost network-scripts]# vim ifcfg-bond-bond0
DEVICE=bond0
BONDING_OPTS=mode=active-backup
TYPE=Bond
BONDING_MASTER=yes
BOOTPROTO=static		# 将 dhcp 改为static
DEFROUTE=yes
PEERDNS=yes
PEERROUTES=yes
IPV4_FAILURE_FATAL=no
IPV6INIT=yes
IPV6_AUTOCONF=yes
IPV6_DEFROUTE=yes
IPV6_PEERDNS=yes
IPV6_PEERROUTES=yes
IPV6_FAILURE_FATAL=no
NAME=bond-bond0
UUID=af2d6662-608c-4f5d-8018-1984cc3d87ef
ONBOOT=yes
IPADDR=192.168.92.20	# 配置 IP 地址
PREFIX=24				# 配置掩码 也可以使用 NETMASK=255.255.255.0
GATEWAY=192.168.92.2	# 配置网关
```

{% note info %}
如果不想修改bond网络接口配置文件可以在第2步的第一条命令后加上 `ip4` "ip地址" `gw4` "网关地址"

	nmcli connection add type bond ifname bond0 mode 1 ip4 192.168.92.20/24 gw4 192.168.92.2
{% endnote %}

### 重启网络，验证配置结果

*查看网卡信息*

```bash
[root@localhost network-scripts]# ip addr show
1: lo:<LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN
link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
inet 127.0.0.1/8 scope host lo
valid_lft forever preferred_lft forever
inet6 ::1/128 scope host
valid_lft forever preferred_lft forever
2: eno16777736:<BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
link/ether 00:0c:29:07:2c:86 brd ff:ff:ff:ff:ff:ff
inet 192.168.92.11/24 brd 192.168.92.255 scope global eno16777736
valid_lft forever preferred_lft forever
inet6 fe80::20c:29ff:fe07:2c86/64 scope link
valid_lft forever preferred_lft forever
3: eno33554960:<BROADCAST,MULTICAST,SLAVE,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast master bond0 state UP qlen 1000
link/ether 00:0c:29:07:2c:90 brd ff:ff:ff:ff:ff:ff
4: eno50332184:<BROADCAST,MULTICAST,SLAVE,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast master bond0 state UP qlen 1000
link/ether 00:0c:29:07:2c:90 brd ff:ff:ff:ff:ff:ff
31: bond0:<BROADCAST,MULTICAST,MASTER,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP
link/ether 00:0c:29:07:2c:90 brd ff:ff:ff:ff:ff:ff
inet 192.168.92.20/24 brd 192.168.92.255 scope global bond0
valid_lft forever preferred_lft forever
inet6 fe80::20c:29ff:fe07:2c90/64 scope link
valid_lft forever preferred_lft forever
```

*查看bond信息*

```bash
[root@localhost network-scripts]# cat /proc/net/bonding/bond0
EthernetChannelBondingDriver: v3.7.1(April27,2011)
BondingMode: fault-tolerance (active-backup)===> bond主备模式
PrimarySlave:None
CurrentlyActiveSlave: eno33554960 ===>当前激活的网卡eno33554960
MII Status: up
MII PollingInterval(ms):100
UpDelay(ms):0
DownDelay(ms):0
SlaveInterface: eno33554960 ===> bond0 组内的网卡
MII Status: up
Speed:1000Mbps
Duplex: full
LinkFailureCount:0
Permanent HW addr:00:0c:29:07:2c:90
Slave queue ID:0
SlaveInterface: eno50332184 ===> bond0 组内的网卡
MII Status: up
Speed:1000Mbps
Duplex: full
LinkFailureCount:0
Permanent HW addr:00:0c:29:07:2c:9a
Slave queue ID:0
```

## 删除 bond 设备

当我们需要删除bond设备的时候,该如何删除呢？请看下面操作

*查看网络设备*

```bash
[root@localhost ~]# ls /sys/class/net/
bond0 bond1 bonding_masters eno16777736 eno33554960 eno50332184 eno67109408 eno83886632 lo
```

*删除bond网络设备*

直接删除 `bond0`，会提示无权限。 

可以通过 `bonding_masters` 文件删除 `bond` 设备, 由于 `bonding_masters` 文件是无法直接修改的，我们需要使用 `echo` 命令进行操作

```bash
[root@localhost ~]# echo -bond0 >/sys/class/net/bonding_masters
```

{% note info %}
`echo` 后面的 `-` 是删除设备，`+` 是增加设备
{% endnote %}