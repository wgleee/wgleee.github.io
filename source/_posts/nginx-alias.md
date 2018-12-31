---
title: NGINX 虚拟目录
date: 2018-12-30 14:48:43
categories: nginx
tags:
  - nginx
---

NGINX 是通过 `alias` 设置虚拟目录，在 NGINX 的配置中， `alias` 指令和 `root` 指令是有区别的

    1. alias 指定的目录是准确的，即 location 匹配访问的 path 目录下的文件直接是在 alias 目录下查找的；
    2. root指定的目录是location匹配访问的path目录的上一级目录,这个 path 目录一定要是真实存在root指定目录下的；
    3. 使用 alias 标签的目录块中不能使用 rewrite 的 break（具体原因不明）；另外， alias 指定的目录后面必须要加上 "/" 符号！
    4. alias 虚拟目录配置中，location匹配的path目录如果后面不带 "/" ，那么访问的url地址中这个path目录后面加不加 "/" 不影响访问，访问时它会自动加上 "/" ； 但是如果location匹配的path目录后面加上 "/"，那么访问的url地址中这个path目录必须要加上 "/"，访问时它不会自动加上 "/"。如果不加上"/"，访问就会失败！
    5. root目录配置中，location 匹配的 path 目录后面带不带 "/"，都不会影响访问。

<!-- more -->

**举例说明（比如 nginx 配置的域名是 wglee.org）**

```
location /huan/ {
    alias /home/www/huan/;
}
```

在上面 `alias` 虚拟目录配置下，访问 `http://wglee.org/huan/a.html` 实际指定的是 `/home/www/huan/a.html`。

> 注意: alias 指定的目录后面必须要加上"/"，即 `/home/www/huan/` 不能改成 `/home/www/huan`

**上面的配置也可以改成 `root` 指令配置，如下**

这样 nginx 就会去 `/home/www/huan` 下寻找 `http://wglee.org/huan/a.html` 资源，两者配置后的访问效果是一样的！

```
location /huan/ {
    root /home/www/;
}
```
