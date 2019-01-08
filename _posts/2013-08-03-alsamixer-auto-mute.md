---
layout: post
title:  alsamixer auto-mute 未关闭，重启后静音
category: system
tags: [alsa, audio]
---

Arch 下的 `alsamixer` 声音设置好后，重启又重新变为静音，好奇怪。

开始还在纠结是 alsa 服务的问题，纯属自扰。杯具在于：

`alsamixer` 中的 `Auto-Mute M` 选项没有 `Disable` 掉，关闭 **自动静音** 就 Ok 了。百文不如一见：

![alsamixer auto-mute disabled][0]

[0]: http://media-cache-ak0.pinimg.com/originals/e6/95/46/e695463439af166ea94f2c32966ea87d.jpg

## 南辕北辙

`systemctl` 查看 `alsa` 服务状态，虽然都是加载了，但状态却是 `dead` 有些猫腻，就接着探究

    · systemctl status alsa-state.service
    alsa-state.service - Manage Sound Card State (restore and store)
       Loaded: loaded (/usr/lib/systemd/system/alsa-state.service; static)
       Active: inactive (dead)

    Aug 03 19:39:07 systemd[1]: Started Manage Sound Card State (restore and store).

    · systemctl status alsa-store.service
    alsa-store.service - Store Sound Card State
       Loaded: loaded (/usr/lib/systemd/system/alsa-store.service; static)
       Active: inactive (dead)

    Aug 03 17:13:49 systemd[1]: Started Store Sound Card State.

    · systemctl status alsa-restore.service
    alsa-restore.service - Restore Sound Card State
       Loaded: loaded (/usr/lib/systemd/system/alsa-restore.service; static)
       Active: inactive (dead) since Sat 2013-08-03 19:39:07 CST; 48min ago
      Process: 271 ExecStart=/usr/bin/alsactl restore (code=exited, status=0/SUCCESS)

    Aug 03 20:52:30 systemd[1]: Starting Restore Sound Card State...

尝试启动服务 `systemctl` 执行 `enable` 提示说，不能安装：

    $ sudo systemctl enable alsa-restore.service
    The unit files have no [Install] section. They are not meant to be enabled using systemctl.
    Possible reasons for having this kind of units are:
    1) A unit may be statically enabled by being symlinked from another unit's
       .wants/ or .requires/ directory.
    2) A unit's purpose may be to act as a helper for some other unit which has
       a requirement dependency on it.
    3) A unit may be started when needed via activation (socket, path, timer,
       D-Bus, udev, scripted systemctl call, ...).

[https://bbs.archlinux.org/viewtopic.php?id=146794][1]

[1]: https://bbs.archlinux.org/viewtopic.php?id=146794

上面这篇帖子提到 `static` 状态是，以及 `alsa store / restore` 服务的链接文件都存在：

    · systemctl list-unit-files --type=service | grep alsa
    alsa-restore.service                        static
    alsa-state.service                          static
    alsa-store.service                          static

     · systemctl is-enabled alsa-store.service alsa-restore.service
     static
     static

[https://bbs.archlinux.org/viewtopic.php?id=147964][2]

[2]: https://bbs.archlinux.org/viewtopic.php?id=147964

然后又科普了一下 `static` 状态：其他 `unit` 依赖该服务时会启动它，或是手动启动

> `static` units are those **which cannot be enabled/disabled**, but it doesn't mean they are always executed.
> They will only if another unit depends on them, or if they are manually started.

从 status 的日志中也可以看出来，开机的时候服务有启动过的。虽然无声和 `systemctl` 没半毛钱关系。
分析的这一陀，绕了弯子，但至少科普了下 `systemctl` 的 `static` 服务状态。



<br/>

本文标题 | [{{ page.title }}]({{ page.url }})
-------- |:--------
原始链接 | <{{ site.url }}{{ page.url }}>
