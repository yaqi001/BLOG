---
author: yingyun001
layout: post
title: "在 Fedora22 上安装 Docker"
date: 2015-09-23 17:08
category: docker
tags:
- docker storage driver
- fedora22
---

## 准备环境

0. 首先准备好 fedora22 操作系统。
   
   我的 fedora22 的内核版本：

   ~~~bash
   # uname -r
   4.0.4-301.fc22.x86_64
   ~~~

1. 更新系统。

   ~~~bash
   # dnf -y update
   ~~~

## 安装

根据[上一篇博客](/docker/2015/09/22/new_apt_and_yum_repos)，我们有两种安装 Docker Engine 的方式：

~~~ bash
$ curl -sSL https://get.docker.com/ | sh
$ sudo service docker start
~~~

或 

~~~ bash
$ cat >/etc/yum.repos.d/docker.repo <<-EOF
[dockerrepo]
name=Docker Repository
baseurl=https://yum.dockerproject.org/repo/main/fedora/22
enabled=1
gpgcheck=1
gpgkey=https://yum.dockerproject.org/gpg
EOF

$ sudo yum install docker-engine
$ sudo service docker start
~~~

> **注意：**
> 如果您发现当您开启 Docker 后台程序的时候，您被告知不具有开启 Docker 的权限，请您[往这看](/linux/2015/09/21/systemd_in_Fedora22/)。

**在您确定您有权限开启 Docker 后台程序的情况下，Docker 仍不能成功开启的话，就需要更改 Docker 的存储驱动了：**

* 让我们看看 docker.service 的配置：

  ~~~ bash
  [helen@zhangyingyun ~]$ cat /usr/lib/systemd/system/docker.service 
  [Unit]
  Description=Docker Application Container Engine
  Documentation=https://docs.docker.com
  After=network.target docker.socket
  Requires=docker.socket

  [Service]
  Type=notify
  ExecStart=/usr/bin/docker daemon -H fd://
  MountFlags=slave
  LimitNOFILE=1048576
  LimitNPROC=1048576
  LimitCORE=infinity

  [Install]
  WantedBy=multi-user.target
  ~~~
  
 现在您可能会问为什么要看 `docker.service` 文件，是在这个文件下更改存储驱动吗？下面我们就来解答这个问题：


## 使用 systemd 来配置并控制 Docker
                  
我们可以使用 systemd 的 drop-in 文件（可以理解为一个临时文件），该文件要放在 `/etc/systemd/system/docker.service.d` 或者 `/etc/systemd/system/docker.server` 目录中。我们需要使用这个 drop-in 文件来自定义 Docker 的后台应用程序选项，这是因为存放在 `/usr/lib/systemd/system` 或者 `/lib/systemd/system` 目录中的文件包含了 Docker 的默认选项且不能被编辑。

但是，如果您之前使用了一个有 EnvironmentFile 配置属性(该属性值通常指向 /etc/sysconfig/docker 文件）的包，那么出于向后兼容的考虑，您应该在 /etc/systemd/system/docker.service.d 目录下创建具有以下内容的文件。

---

0. 让我们来为 Docker systemd 单元创建 drop-in 文件的目录：

   ~~~ bash
   mkdir -p /etc/systemd/system/docker.service.d
   ~~~

1. 该目录下的文件名可以任取，且可以有多个 drop in 文件。

   ~~~ bash
   vi /etc/systemd/system/docker.service.d/enable-options.conf
   [Service]
   EnvironmentFile=-/etc/sysconfig/docker
   ExecStart=
   ExecStart=/usr/bin/docker -d -H fd:// $OPTIONS
   ~~~

   > **注意**：这里有两行以 **ExecStart** 开头的配置信息，这是用于重写 *ExecStart* 的。

2. 接下来我们要在 `EnvironmentFile` 指向的 `/etc/sysconfig/docker` 文件中配置 **OPTIONS** 来调整 docker 后台程序的存储驱动了。

   ~~~ bash
   OPTIONS="-s overlay"
   ~~~ 

3. 让我们检查一下配置是否生效。

   ~~~ bash
   [root@zhangyingyun ~]# systemctl status docker
   ● docker.service - Docker Application Container Engine
   Loaded: loaded (/usr/lib/systemd/system/docker.service; disabled; vendor preset: disabled)
   Active: failed (Result: exit-code) since 五 2015-09-25 14:10:16 CST; 2h 44min ago
   Docs: https://docs.docker.com
   Process: 5627 ExecStart=/usr/bin/docker daemon -H fd:// (code=exited, status=2)
   Main PID: 5627 (code=exited, status=2)
   CGroup: /system.slice/docker.service

   9月 25 14:08:45 zhangyingyun.eayun systemd[1]: Starting Docker A...
   9月 25 14:10:15 zhangyingyun.eayun systemd[1]: docker.service st...
   9月 25 14:10:16 zhangyingyun.eayun systemd[1]: docker.service: m...
   9月 25 14:10:16 zhangyingyun.eayun systemd[1]: Failed to start D...
   9月 25 14:10:16 zhangyingyun.eayun systemd[1]: Unit docker.servi...
   9月 25 14:10:16 zhangyingyun.eayun systemd[1]: docker.service fa...
   9月 25 16:53:46 zhangyingyun.eayun systemd[1]: Stopped Docker Ap...
   9月 25 16:53:53 zhangyingyun.eayun systemd[1]: Stopped Docker Ap...
   ** Warning: docker.service changed on disk. Run 'systemctl daemon-reload' to reload units. **
   Hint: Some lines were ellipsized, use -l to show in full.
   ~~~

   根据上面的**提示**，修改了 `docker.service` 文件后必须重新加载后台程序单元才能真正生效：

4. 重新加载 Docker 的后台程序

   ~~~ bash
   # systemctl daemon-reload
   [root@zhangyingyun ~]# systemctl status docker
   ● docker.service - Docker Application Container Engine
     Loaded: loaded (/usr/lib/systemd/system/docker.service; disabled; vendor preset: disabled)
    Drop-In: /etc/systemd/system/docker.service.d
             └─enable-options.conf
     Active: failed (Result: exit-code) since 五 2015-09-25 14:10:16 CST; 2h 47min ago
        Docs: https://docs.docker.com
   Main PID: 5627 (code=exited, status=2)

   9月 25 14:08:45 zhangyingyun.eayun systemd[1]: Starting Docker A...
   9月 25 14:10:15 zhangyingyun.eayun systemd[1]: docker.service st...
   9月 25 14:10:16 zhangyingyun.eayun systemd[1]: docker.service: m...
   9月 25 14:10:16 zhangyingyun.eayun systemd[1]: Failed to start D...
   9月 25 14:10:16 zhangyingyun.eayun systemd[1]: Unit docker.servi...
   9月 25 14:10:16 zhangyingyun.eayun systemd[1]: docker.service fa...
   9月 25 16:53:46 zhangyingyun.eayun systemd[1]: Stopped Docker Ap...
   9月 25 16:53:53 zhangyingyun.eayun systemd[1]: Stopped Docker Ap...
   Hint: Some lines were ellipsized, use -l to show in full.
   ~~~

   虽然 Docker 仍未开启，但是可以从以上内容看出 systemd 已经检测到了 drop-in 文件。

5. 重启 Docker：
   
   ~~~bash
   # systemctl restart docker
   ~~~

## Docker 存储驱动

1. AUFS 
现在您就可以开启您的 Docker 之旅啦！
