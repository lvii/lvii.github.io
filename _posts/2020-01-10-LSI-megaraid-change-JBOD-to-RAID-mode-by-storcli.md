---
layout: post
title: "使用 storcli 将硬盘从 JBOD 模式改为 RAID 模式"
category: system
tags: [RAID]
---

# WHAT

给一台服务器增加 SSD 后，发现新加的 2 块 SSD 硬盘直接被系统识别

我都还没来的及配置 RAID 竟然被识别，一看发现新加的 SSD 是 JBOD 模式：

    # storcli /c0/e252/s6 show
    Controller = 0
    Status = Success
    Description = Show Drive Information Succeeded.

    Drive Information :
    =================

    ----------------------------------------------------------------------------
    EID:Slt DID State DG       Size Intf Med SED PI SeSz Model               Sp
    ----------------------------------------------------------------------------
    252:6    17 JBOD  -  893.137 GB SATA SSD N   N  512B INTEL SSDSC2KB960G7 U
    ----------------------------------------------------------------------------

# HOW

将物理盘 `set good` 设置一下，即可从 JBOD 模式切换回 RAID 模式：

    # storcli /c0/e252/s6 set good
    Controller = 0
    Status = Failure
    Description = Set Drive Good Failed.

    Detailed Status :
    ===============

    -----------------------------------------------------------------------
    Drive       Status  ErrCd ErrMsg
    -----------------------------------------------------------------------
    /c0/e252/s6 Failure   255 Drive is JBOD! Use -force option to confirm.
    -----------------------------------------------------------------------

需要增加 `force` 参数：

    # storcli /c0/e252/s6 set good force
    Controller = 0
    Status = Success
    Description = Set Drive Good Succeeded.

执行成功后磁盘 `State` 状态从 `JBOD` 变为 `UGood` 然后就可以配置 RAID 了：

    # storcli /c0/e252/s6 show
    Controller = 0
    Status = Success
    Description = Show Drive Information Succeeded.

    Drive Information :
    =================

    ----------------------------------------------------------------------------
    EID:Slt DID State DG       Size Intf Med SED PI SeSz Model               Sp
    ----------------------------------------------------------------------------
    252:6    17 UGood  1 893.137 GB SATA SSD N   N  512B INTEL SSDSC2KB960G7 U
    ----------------------------------------------------------------------------

<br/>

本文标题 | [{{ page.title }}]({{ page.url }})
-------- |:--------
原始链接 | <{{ site.url }}{{ page.url }}>
