---
layout: post
title: KDE 升级之后 taskbar 丢失
category: system
tags: [kde]
---

gentoo update 了一下 world 后，重启进入 kde 发现状态栏不见了  
添加了一个新的 panel 后，发现有些问题：

- 新的 panel 使用的默认的配置，之前的设置都没了
- notify 提示的时候，竟然莫名的弹出 2 个提示框

这个就比较杯具了 gtalk，irc 和脚本的提示同时弹出两个框，太 囧rz 了

KDE 每次升级，都会出现些杯具事件。在没有找到更好的 DE 只能继续忍受它

牢骚完了，找到底是哪个配置文件出了问题 grep 一下 panel 和 taskbar 两个关键词：

    · grep panel ~/.kde4/share/config/*
    ~/.kde4/share/config/dolphinrc:LockPanels=false
    ~/.kde4/share/config/plasma-desktop-appletsrc:plugin=panel
    ~/.kde4/share/config/plasma-desktoprc:panelVisibility=0

    · grep taskbar ~/.kde4/share/config/*
    ~/.kde4/share/config/kdeglobals:taskbarFont=Sans Serif,9,-1,5,50,0,0,0,0,0
    ~/.kde4/share/config/ktelepathy.notifyrc:Action=Sound|Taskbar
    ~/.kde4/share/config/kwinrc:kwin4_effect_taskbarthumbnailEnabled=false
    ~/.kde4/share/config/kwinrc:InactiveTabsSkipTaskbar=false

应该和 `plasma-desktoprc` 文件有点关系：

这段配置对应的是，之前新添加的 panel

    [Containments][44]
    activity=
    activityId=
    desktop=-1
    formfactor=2
    geometry=0,-93,1280,27
    immutability=1
    lastDesktop=-1
    lastScreen=0
    location=3
    plugin=panel
    screen=0
    zvalue=0

这段是老的 panel 配置文件：

    [Containments][1]
    ActionPluginsSource=Global
    activity=New Activity
    activityId=881e6c57-204f-442b-975c-6f7cdbd49ac8
    desktop=-1
    formfactor=2
    geometry=0,-33,1280,27
    immutability=1
    lastDesktop=-1
    lastScreen=0
    location=4
    plugin=panel
    screen=-1    ## XXX 之前有外接过显示器
    zvalue=0

对比了一下 `screen` 这个字段配置有些不一样。将老的 panel 的 screen 字段改为 `0`

删除刚才增加的 panel 重新登录了一下，之前的状态栏回归鸟。。。

按理说 kde 升级应该不会去改这个字段啊

之前有插过外接显示器，不知道是否是那时候埋下的坑

但是为什么这次升级才掉坑里，百思不得其解 - -.







<br/>

本文标题 | [{{ page.title }}]({{ page.url }})
-------- |:--------
原始链接 | <{{ site.url }}{{ page.url }}>
