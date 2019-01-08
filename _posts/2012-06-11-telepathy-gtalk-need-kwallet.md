---
layout: post
title: telepathy 升级后 无法登录 gtalk
category: soft
tags: [gtalk, kwallet, kde]
---

arch 升级后 telepathy 无法登录 gtalk 。打开 telepathy 后登录 gtalk 老是提示 kwallet 输入密码。
输入之前设置过的密码没用，我在 **系统设置** 里面也没有找到和 kwallet 相关的设置选项。
升级之前没有安装 kwallet 是 OK 的，为何现在要调用它来管理密码。就想删除 telepathy 使用 kwallet 认证的模块试试：

    # pacman -Ss telepathy
    ...
    extra/telepathy-kde-auth-handler 0.3.1-1 (kde-telepathy) [installed]
        Provide UI/KWallet Integration For Passwords and SSL Errors on Account Connect
    ...
    # pacman -Rsun telepathy-kde-auth-handler
        removed telepathy-kde-auth-handler (0.3.1-1)
    ...

删除之后，重新创建 gtalk 帐号，貌似还是没啥用，还是会弹出 kwallet 提示。
google 发现这个兄台也遇到类似的事情，新版本加入了 kwallet 依赖，但升级时没有自动安装依赖

[Auth failure with telepathy-kde](http://forum.kde.org/viewtopic.php?f=18&t=101712#p220764)

参考人家的解决方法，安装 kwallet 呗：

    # pacman -S --need kdeutils-kwallet telepathy-kde-auth-handler

安装后 telepathy 启动输入 kwallet 密码。
要反复输入 3 次，它才罢休，竟然还提示说我错鸟 ... 这个太纠结了。
更纠结的是，在第二次输入之后 gtalk 几经连接上了，竟然弹出来骚扰人，果断关掉。

从此，每次登录 gtalk 就有了输入 kwallet 密码的后遗症 ...




<br/>

本文标题 | [{{ page.title }}]({{ page.url }})
-------- |:--------
原始链接 | <{{ site.url }}{{ page.url }}>
