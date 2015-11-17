---
author: yingyun001
layout: post
title: "Fedora22 上的 systemd：不具有重启服务的权限"
date: 2015-09-21 9:08
category: linux
tags:
- systemd
- fedora22
---

* 原文：[systemd in Fedora 22: Failed to restart service: Access Denied](https://major.io/2015/09/18/systemd-in-fedora-22-failed-to-restart-service-access-denied/) 

# Fedora 22 之 systemd：没有权限开启服务

如果您正在 Fedora 22 下工作，那么您的 systemd 的版本已经更新到了 [systemd-219-24.fc22](https://bodhi.fedoraproject.org/updates/FEDORA-2015-15821)，您可能会遇到这个问题：

~~~ bash
# systemctl status firewalld
Failed to get properties: Access denied
~~~


查看您的审计日志（`/var/log/audit/audit.log`）可能是这样的：

~~~ bash
type=SERVICE_START msg=audit(1443110164.324:96): pid=1 uid=0 auid=4294967295 ses=4294967295 subj=system_u:system_r:init_t:s0 msg='unit=chronyd comm="systemd" exe="/usr/lib/systemd/systemd" hostname=? addr=? terminal=? res=success'
type=SERVICE_START msg=audit(1443110164.338:97): pid=1 uid=0 auid=4294967295 ses=4294967295 subj=system_u:system_r:init_t:s0 msg='unit=spice-vdagentd comm="systemd" exe="/usr/lib/systemd/systemd" hostname=? addr=? terminal=? res=success'
type=SERVICE_START msg=audit(1443110164.457:98): pid=1 uid=0 auid=4294967295 ses=4294967295 subj=system_u:system_r:init_t:s0 msg='unit=systemd-logind comm="systemd" exe="/usr/lib/systemd/systemd" hostname=? addr=? terminal=? res=success'
type=SERVICE_START msg=audit(1443110165.741:99): pid=1 uid=0 auid=4294967295 ses=4294967295 subj=system_u:system_r:init_t:s0 msg='unit=abrtd comm="systemd" exe="/usr/lib/systemd/systemd" hostname=? addr=? terminal=? res=success'
~~~



这个 bug 还在修复中，但是还有一个变通的解决方案就是您可以用下面的这个命令重新执行 systemd。
`systemctl daemon-reexec`

然后您就可以关闭，启动并重启服务了。您还可以更改重启和关机的运行级别。

向 [Kevin Fenzi](https://fedoraproject.org/wiki/User:Kevin) 的 workaround 致敬！















































