---
title: CentOS 6.x 配置 bond
date: 2018-12-31 10:24:50
categories: linux
tags:
  - 双网卡绑定
---

`CentOS 6` 下使用双网卡配置 `bond0`, `CentOS 6` `bond` 配置不需要在 `/etc/modprobe` 中定义 `bond` 直接在网卡中定义 `BONDING_OPTS` 即可,例如 bond0: `BONDING_OPTS="mode=0 miimon=100"`

<!-- more -->

## 物理网卡 eth0 eth1 绑定为 bond0 

### 物理网卡配置

*ifcfg-eth0*

```bash
DEVICE=eth0
ONBOOT=yes
BOOTPROTO=none
MASTER=bond0
SLAVE=yes
```

*ifcfg-eth1*

```bash
DEVICE=eth1
ONBOOT=yes
BOOTPROTO=none
MASTER=bond0
SLAVE=yes
```

### bond0 网卡配置

*ifcfg-bond0*

```bash
DEVICE=bond0
ONBOOT=yes
BOOTPROTO=none
IPADDR=xx.xx.xx.xx
NETMASK=xx.xx.xx.xx
GATEWAY=xx.xx.xx.xx
DNS1=xx.xx.xx.xx
BONDING_OPTS="mode=0 miimon=100"
```

*重启网卡*

```bash
service network restart
```

### 查看 bond0 网卡状态

```bash
[root@localhost network-scripts]# cat /proc/net/bonding/bond0
EthernetChannelBondingDriver: v3.7.1(April27,2011)
BondingMode: load balancing (round-robin)
MII Status: up
MII PollingInterval(ms):100
UpDelay(ms):0
DownDelay(ms):0
SlaveInterface: eth1
MII Status: up
Speed:1000Mbps
Duplex: full
LinkFailureCount:0
Permanent HW addr:00:0c:29:a1:34:eb
Slave queue ID:0
SlaveInterface: eth2
MII Status: up
Speed:1000Mbps
Duplex: full
LinkFailureCount:0
Permanent HW addr:00:0c:29:a1:34:e1
Slave queue ID:0
```

{% note info %}
日志可以使用 `tailf /var/log/messages` 实时查看
{% endnote %}