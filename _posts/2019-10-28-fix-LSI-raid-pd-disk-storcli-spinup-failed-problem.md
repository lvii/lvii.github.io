---
layout: post
title: "使用 storcli 修复 LSI RAID 硬盘 spinup failed 问题"
category: hardware
tags: [RAID]
---

# WHAT

最近处理一批 LSI RAID 故障的机器，遇到硬盘 `UGood` 状态竟无法配置 RAID 的情况

发现 PD 物理盘的 Spin 状态是 `D` down 状态：

    $ sudo /usr/local/sbin/storcli /c0/e0/s8 show
    Controller = 0
    Status = Success
    Description = Show Drive Information Succeeded.

    Drive Information :
    =================

    --------------------------------------------------------------------------
    EID:Slt DID State DG     Size Intf Med SED PI SeSz Model               Sp
    --------------------------------------------------------------------------
    0:8       4 UGood -  7.276 TB SATA HDD N   N  512B ST8000NM0016-1U3101 D  <-- Spin 状态为 D
    --------------------------------------------------------------------------

使用 `storcli` 设置 PD Spin 状态 `spinup` 结果报错：

    $ sudo /usr/local/sbin/storcli /c0/e0/s8 spinup
    Controller = 0
    Status = Failure
    Description = Spin Up Drive Failed.

    Detailed Status :
    ===============

    -----------------------------------------------------------------------
    Drive     Status  ErrCd ErrMsg
    -----------------------------------------------------------------------
    /c0/e0/s8 Failure    50 device state doesn't support requested command
    -----------------------------------------------------------------------

# HOW

使用 `erase` 清除一下 PD 硬盘 **状态** ：

    $ sudo storcli /c0/e0/s8 help|grep -w erase
    storcli /cx[/ex]/sx start erase [simple| normal| thorough | standard| threepass | crypto]
    storcli /cx[/ex]/sx stop erase
    storcli /cx[/ex]/sx show erase

硬盘 Spin 状态从 `D` 变为 `U` ：

    $ sudo storcli /c0/e0/s8 start erase simple

earse 除了擦除 PD 状态，还会擦除数据，8T 硬盘耗时还是比较久的：

    $ sudo storcli /c0/e0/s8 show erase
    Controller = 0
    Status = Success
    Description = Show Drive Erase Status Succeeded.

    ----------------------------------------------------
    Drive-ID  Progress% Status      Estimated Time Left
    ----------------------------------------------------
    /c0/e0/s8         1 In progress 9 Hours 31 Minutes
    ----------------------------------------------------

Spin 状态复位后，停止擦除即可：

    $ sudo storcli /c0/e0/s8 stop erase
    Controller = 0
    Status = Success
    Description = Stop Drive Erase Succeeded.

清除磁盘状态后，Spin 状态可以顺利 `spinup` 和 `spindown` 开关：

    $ sudo storcli /c0/e0/s8 spindown
    Controller = 0
    Status = Success
    Description = Spin Down Drive Succeeded.

    $ sudo storcli /c0/e0/s8 show
    Controller = 0
    Status = Success
    Description = Show Drive Information Succeeded.

    Drive Information :
    =================

    --------------------------------------------------------------------------
    EID:Slt DID State DG     Size Intf Med SED PI SeSz Model               Sp
    --------------------------------------------------------------------------
    0:8       4 UGood -  7.276 TB SATA HDD N   N  512B ST8000NM0016-1U3101 D
    --------------------------------------------------------------------------

    $ sudo storcli /c0/e0/s8 spinup
    Controller = 0
    Status = Success
    Description = Spin Up Drive Succeeded.

    $ sudo storcli /c0/e0/s8 show
    Controller = 0
    Status = Success
    Description = Show Drive Information Succeeded.

    Drive Information :
    =================

    --------------------------------------------------------------------------
    EID:Slt DID State DG     Size Intf Med SED PI SeSz Model               Sp
    --------------------------------------------------------------------------
    0:8       4 UGood -  7.276 TB SATA HDD N   N  512B ST8000NM0016-1U3101 U
    --------------------------------------------------------------------------

故障盘设置 `offline`、`missing` 下线后，需要设置 PD `spindown` ：

    storcli /cx/ey/sz set offline
    storcli /cx/ey/sz set missing
    storcli /cx/ey/sz spindown

硬盘在 `spindown` 状态后是无法创建 RAID 的：

    $ sudo storcli /c0/e0/s8 spindown
    Controller = 0
    Status = Success
    Description = Spin Down Drive Succeeded.

    $ sudo storcli /c0 add vd type=r0 drives=0:8
    Controller = 0
    Status = Failure
    Description = physical disk does not have appropriate attributes

重新设为 `spinup` 后，`UGood` 硬盘就可以正常创建 RAID 了：

    $ sudo storcli /c0/e0/s8 spinup
    Controller = 0
    Status = Success
    Description = Spin Up Drive Succeeded.

    $ sudo storcli /c0/e0/s8 show
    Controller = 0
    Status = Success
    Description = Show Drive Information Succeeded.

    Drive Information :
    =================

    --------------------------------------------------------------------------
    EID:Slt DID State DG     Size Intf Med SED PI SeSz Model               Sp
    --------------------------------------------------------------------------
    0:8       4 UGood -  7.276 TB SATA HDD N   N  512B ST8000NM0016-1U3101 U
    --------------------------------------------------------------------------

    $ sudo storcli /c0 add vd type=r0 drives=0:8
    Controller = 0
    Status = Success
    Description = Add VD Succeeded

# reference

- [storcli](https://datahunter.org/storcli)
- [LSI storcli64 examples](https://support.dvsus.com/hc/en-us/articles/115000636106-LSI-storcli64-examples)
- [How do I replace a failed drive with LSI 9280 cards?](https://www.45drives.com/wiki/index.php?title=How_do_I_replace_a_failed_drive_with_LSI_9280_cards%3F)
- [How do I remove a failing disk from a LSI MegaRAID disk group?](https://serverfault.com/questions/954302/how-do-i-remove-a-failing-disk-from-a-lsi-megaraid-disk-group)

<br/>

本文标题 | [{{ page.title }}]({{ page.url }})
-------- |:--------
原始链接 | <{{ site.url }}{{ page.url }}>
