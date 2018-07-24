---
title: RPM 包制作神器 multipkg
date: 2018-03-14 15:56:14
categories: linux
tags: rpm
---

> 在生产环境中大多数软件包都会采用编译的方式进行安装，编译安装软件，优点是可以定制化安装目录、按需开启功能等，缺点是需要查找并实验出适合的编译参数，诸如MySQL之类的软件编译耗时过长。 如果做成 rpm 然后在搭建一套内部的使用的 yum 仓库就会方便很多。但传统制作 rpm 包的方式但过于复杂难用。本文将介绍一款简单易用 rpm 打包工具 `multipkg`

<!-- more -->

## 安装 multipkg

- [multipkg 源代码仓库](https://github.com/ytoolshed/multipkg.git)
- [参考资料](https://yq.aliyun.com/articles/68346)

```bash
git clone https://github.com/ytoolshed/multipkg.git

cd multipkg

yum install perl-YAML-Syck perl-ExtUtils-MakeMaker

PREFIX=./root PKGVERID=0 INSTALLDIR=source scripts/transform
perl -I ./source/lib root/usr/bin/multipkg -t .

sudo yum -y install multipkg-*rpm

rm multipkg*rpm

git-multipkg -b https://github.com/ytoolshed/ multipkg

sudo yum upgrade ./multipkg*rpm
```

## 示例

> [示例代码](https://github.com/liwanggui/multipkg-examples.git)

以 nginx 为例

**nginx 工程目录结构**

```bash
nginx/
├── index.yaml
├── nginx-1.3.10.tar.gz
├── root
│   └── usr
│       └── local
│           └── nginx
│               ├── conf
│               │   ├── enable-ssl-vhost-example.conf
│               │   ├── enable-php.conf
│               │   ├── nginx.conf
│               │   ├── pathinfo.conf
│               │   ├── proxy.conf
│               │   └── vhost
│               └── support-file
│                   └── nginx
└── scripts
    ├── build
    ├── post.sh
    └── postun.sh
```

**目录解释**

- root: root 目录中的文件直接提供了 rpm 文件列表和安装路径，目录内的文件会自动加入到生成的 rpm 包中.
- scripts: multipkg 生成 rpm 过程中需要使用的脚本及 rpm 安装前后，卸载前后使用的脚本
  - build: 源码编译安装相关命令
  - pre.sh: rpm 包安装前
  - post.sh: rpm 包安装后
  - preun.sh: rpm 包卸载前
  - postun.sh: rpm 包卸载后

**生成 rpm 包**

在工程目录执行以下命令生成 rpm 包

```bash
[root@localhost nginx]# multipkg .
nginx-1.3.10-0.1521267394.x86_64.rpm
```
