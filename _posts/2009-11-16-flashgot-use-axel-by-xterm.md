---
layout: post
title: firefox 插件 flashgot 通过 xterm 调用 axel 下载
category: soft
tags: [firefox, plugin, xterm]
---

firefox 安装 flashgot 插件调用 linux 下的 `axel` 下载资源

对于 cli 工具需要通过 **终端** 来调用 `axel` 完成下载

使用 `xterm -e <exec_command>` 打开一个 xterm 终端来调用 cli 工具

要在 flashgot 中通过 xterm 调用 `axel` 下载资源，设置如下图所示：

![flashgot setting](http://fc05.deviantart.net/fs51/f/2009/319/6/2/firefox_by_57lvii.png)

即可弹出 xterm 终端，执行 `axel` 下载相应资源

**PS: 传说 `axel` 不怎么更新，而且对于 https 下载支持不好，可以换用** `aria2rc`




<br/>

本文标题 | [{{ page.title }}]({{ page.url }})
-------- |:--------
原始链接 | <{{ site.url }}{{ page.url }}>
