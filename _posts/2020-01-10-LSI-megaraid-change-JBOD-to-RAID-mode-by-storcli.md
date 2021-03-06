---
layout: post
title: "使用 storcli 将 LSI RAID 硬盘从 JBOD 模式改为 RAID 模式"
category: hardware
tags: [RAID]
---

# WHAT

<https://en.wikipedia.org/wiki/Non-RAID_drive_architectures>

> JBOD (abbreviated from "**Joint Batch Of Disks**" or colloquially "**just a bunch of disks/drives**") is an architecture using multiple hard drives exposed as individual devices. Hard drives may be treated **independently** or may be combined into one or more **logical volumes** using a volume manager like **LVM** or **mdadm**; such volumes are usually called "**spanned**" or "**linear \| SPAN \| BIG**".
>
> What makes a SPAN or BIG different from RAID configurations is the possibility for the selection of drives. While RAID usually requires all drives to **be of similar capacity** and it is preferred that the same or similar drive models are used for performance reasons, **a spanned volume does not have such requirements**.

多块盘组合的 JBOD 模式，类似 LVM 顺序拼接，对拼接的磁盘大小没有要求。而 RAID 模式对磁盘大小有限制，会选择 **容量最小（木桶原理）** 的硬盘进行总容量的计算，大盘多余的容量将被忽略。不同容量的盘做 RAID5 就是这样处理的。

厂里一台服务器增加 SSD 后，发现新加的 2 块 SSD 硬盘直接被系统识别了。

可都还没来的及配置 RAID 竟然被识别，一看发现新加的 SSD 是 `JBOD` 状态：

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
                ^^^^

# HOW

文档也没找到明确的 **切换** JBOD 和 RAID 模式的命令，只有设置硬盘 **状态** 的命令：

command | operation
------- | ---------
`storcli /cx[/ex]/sx set jbod` | This command sets the *drive state* to **JBOD**.
`storcli /cx[/ex]/sx set good [force]` | This drive changes the *drive state* to **Unconfigured Good**.

将物理盘设置 `set good` 一下：

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

追加 `force` 参数：

    # storcli /c0/e252/s6 set good force
    Controller = 0
    Status = Success
    Description = Set Drive Good Succeeded.

执行成功后，磁盘 `State` 状态从 `JBOD` 变为 `UGood` 就可以正常配置 RAID 了：

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

**单块** 硬盘的 `JBOD` 状态来区分 RAID 模式，而且 RAID 和 JBOD 是可以 **同时混用** 的。

RAID 卡设置里有是否启用 **JBOD 模式** 的 **全局** 开关：

    # storcli /c0 help|grep -i jbod=
    storcli /cx set jbod=<on|off> [force]

    # storcli /c0 show all|grep -i jbod
    Support JBOD = Yes
    Support SecurityonJBOD = Yes
    Support JBOD Write cache = No
    Enable JBOD = No                    <-- 全局开关状态

    # storcli /c0 show jbod
    Controller = 0
    Status = Success
    Description = None

    Controller Properties :
    =====================

    ----------------
    Ctrl_Prop Value
    ----------------
    JBOD      OFF
    ----------------


<br/>

本文标题 | [{{ page.title }}]({{ page.url }})
-------- |:--------
原始链接 | <{{ site.url }}{{ page.url }}>
