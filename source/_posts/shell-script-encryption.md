---
title: SHELL 脚本加密
date: 2017-12-08 16:04:57
categories: bash
tags:
    - linux
    - bash
---

> 在日常工作中经常会写一些脚本来提高工作效率（懒惰是美德），但脚本中难免会有一些敏感信息，例如数据库密码，服务器账号密码等。如何才能有效隐藏或者加密脚本使之成为密文以提高安全性呢？通过在网上搜索实践终于找到了解决方法，下面就介绍 shell 脚本 如何加密。（个人认为比较实用的，当然还有其他方法）

<!-- more -->

{% asset_img example.png 示例 %}

通过上图脚本中的前部分代码我们可以知道脚本将以 "__END__" 字符开头的行之后所有行的内容通过管道传输给你 `tar` 程序进行解压，最后在运行解压出来的脚本。

> Tips: 这个脚本文件是不可以更改的，更改后将无法正常执行。

下面举例说明

*第一步：我们写一个简单的脚本 `ip.sh` (获取本机的公网的 IP)*

```
#!/bin/bash

curl ifconfig.me
```

第二步：写一个对外的脚本 `test.sh`

```
#!/bin/bash

run="ip.sh"

main() {
    LINE=$(awk '/^_END_/ {print NR + 1}' "$0")
    BASE_DIR=$(mktemp -d)

    tail -n +"$LINE" $0 | tar xzm -C $BASE_DIR
    cd $BASE_DIR
    /bin/sh $1
    /bin/rm -rf $BASE_DIR
    exit
}

main $run

_END_
```

第三步： 使用 `tar` 工具将 `ip.sh` 压缩并追加到 `test.sh` 脚本文件中

```
tar czm ip.sh >> test.sh
```

{% asset_img test.png 最终结果 %}

最后我们就得到了一个简单并加密脚本文件了。正常执行下看看结果：

```
# sh test.sh
183.63.xxx.xxx
```

