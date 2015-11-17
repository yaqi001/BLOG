---
author: yingyun001
layout: post
title: "安装 proxychains-ng"
date: 2015-10-20 14:24
category: Tools
tags:
- 代理
- proxychains
---

1. 在终端中输入 `git clone https://github.com/rofl0r/proxychains-ng.git`

2. 进入到 proxychains-ng 目录中，执行：`./configure`

3. (1) C 语言进行编译，需要执行：
       ```
       $ sudo make install-config （安装至系统中）
       ```

   (2) 或者执行 `sudo make`，不安装至系统。

4. 修改配置文件，添加代理设置。
   如果您使用的是 3(1) 这种方式安装的 proxychains，那么其配置文件在：`/usr/local/etc/proxychains.conf`

   ~~~
   在配置文件的最后一行添加您的代理，我的是 socks5  172.16.1.40 7071。
   ~~~

   如果您采用的是 3(2) 这个方式安装的 proxychains，那么其配置文件在源码包下的 src 目录中

   ~~~bash
   ./proxychains4 -f src/proxychains.conf ping google.com
   ~~~
