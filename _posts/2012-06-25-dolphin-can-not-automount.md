---
layout: post
title: kde 下 dolphin 无法自动挂载 u 盘
category: soft
tags: [slim, dolphin, kde]
---

上周升级系统后，发现 kde 下 dolphin 无法自动挂载 U 盘，挂载时提示如下报错：

    An error occurred while accessing 'Removable Media' the system responded:
    An unspecified error has occurred : Not Authorized

google 错误信息，搜到这篇贴子：[KDE mount USB device fails](https://bbs.archlinux.org/viewtopic.php?id=136681)

提到 3 点:

1. 现在用的用户没有在 `storage` **用户组**，但是升级之前都 OK 的，这个应该不是关键
2. `/usr/share/polkit-1/actions/org.freedesktop.udisks.policy` 文件  
   但是帖子里面没有给出有用的线索为什么是它惹的祸，我也没动它
3. `ck-launch-session` 配置选项  
   `slim.conf` 和 `~/.xinitrc` 文件都可以在启动桌面管理器 session 前，启用该进程 ( fcitx 也会依赖它)

帖子中看到了这段提示：

> Note: slim is **ConsoleKit** capable since version `1.3.3`.
> Unless you happen to run an old version, you must no longer include `ck-launch-session`
> in your `.xinitrc` or `slim.conf` login_cmd.

原来 slim `1.3.3` 之后的版本默认已经 **内置启用** ***ConsoleKit***

不需要再手动在 `slim.conf` 或 `~/.xinitrc` 指定 `ck-launch-session` 配置

[slim archwiki](https://wiki.archlinux.org/index.php/SLiM) 中也在 NOTE 中标记了这个变化

之后在 `~/.xinitrc` 中去掉 `ck-launch-session` 后，启动后 dolphin 可以自动挂载了













<br/>

本文标题 | [{{ page.title }}]({{ page.url }})
-------- |:--------
原始链接 | <{{ site.url }}{{ page.url }}>
