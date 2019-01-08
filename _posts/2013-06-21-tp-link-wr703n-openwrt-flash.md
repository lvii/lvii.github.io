---
layout: post
title: TP Link wr703n 刷机 (openwrt) 不完全手册
category: hardware
tags: [openwrt, route]
---

## OPENWRT vs DD-WRT

- openwrt 2013-04-25 发布最新的稳定版本：12.09 Attitude Adjustment (官网主页)
- dd-wrt 针对 wr703n 可用的还是开发版本：v24 preSP2 (Build 21061)

对比 DIY 折腾的文章以及官方 wiki 最终选择刷 openwrt 同时还发现好多刷成砖的杯具案例

## 为何会刷机变砖

- 没有参考官方 wiki 参考像我写的这种山寨刷机教程，杯具了，，，-_-z
- 改过 RAM 和 FLASH 的话，官方固件无法 100% 支持，可能需要第三方固件
- wr703n 从 2011 年上市到现在，不同生产批次的硬件版本，对应不同的 firmware

## 刷机步骤

看官方 wiki ：[http://wiki.openwrt.org/toh/tp-link/tl-wr703n][1] **注意** 以下几个步骤：

- 确认 wr703n 出厂对应的硬件版本，需要登录到原厂管理后台查看：

```
    当前软件版本：3.14.4 Build 120925 Rel.33144n  
    当前硬件版本：WR703N v1 00000000
```

> in the Chinese webadmin interface: "Build 120925" correspond to a v1.7 firmware
> on the internal sticker located on the Ethernet jack (may have 12B042)
> DO NOT RELY ON THE VERSION GIVEN BY THE EXTERNAL STICKER ON CASE BOTTOM :
> it may report falsely "1.6", even if the firmware is actually a V1.7

`120925` 对应的是 v1.7 版本，**不是** 路由器上面贴的条形码上标记的 `v1.6`

根据 wiki 的版本对应表格 v1.7 版本需要使用 openwrt 12.09 (AA) 稳定版本

- 确认好固件版本，下载后的文件名太长需要 **重命名** ：

> Note that the factory default web interface won't accept a file with a long name.
> Rename it to openwrt.bin and you won't get a "23002 Error". 

    mv -iv openwrt-ar71xx-generic-tl-wr703n-v1-squashfs-factory.bin openwrt.bin


## 刷好之后

刷机完成后参考这篇 wiki 进行初始化配置：[http://wiki.openwrt.org/doc/howto/firstlogin][2]

- 设置密码
- 启用 dropbear ssh 服务

openwrt 12.09 版本，已经内置 Luci 设置密码后，浏览器登录 Luci 开始折腾吧

[1]: http://wiki.openwrt.org/toh/tp-link/tl-wr703n
[2]: http://wiki.openwrt.org/doc/howto/firstlogin



<br/>

本文标题 | [{{ page.title }}]({{ page.url }})
-------- |:--------
原始链接 | <{{ site.url }}{{ page.url }}>
