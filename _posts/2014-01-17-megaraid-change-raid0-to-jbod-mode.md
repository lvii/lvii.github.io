---
layout: post
title: "将 megaraid 卡磁盘 raid0 改为 JBOD 模式"
category: system
tags: [raid]
---

## 起因

厂里几台 hadoop 应用是 dell R720 的机器，数据盘默认配置的是 **单盘 raid0** 。磁盘报修的时候，想通过盘符查出对应磁盘的 SN 无从下手。本来想指望 `smartcl -d megaraid,N <device>` 参数查看做过 raid0 磁盘的 SMART 信息。结果让观众失望了，指定的 **盘符** 根本没有半毛钱用，只和 slot ID 有关。应用日志报错显示的是盘符，要再去找 slot ID 和 **盘符** 的对应关系，，，得掰着指头数，一句话点评：反人类

## 经过

在一次更换 SSD 后发现，新换的 SSD 没有做 raid0 操作，直接就被系统识别到了，在 raid 配置界面看到 SSD 磁盘的状态是 `Non-RAID` 这个就是 megaraid 的 `JBOD Mode` **维基百科** 对 JBOD 解释 Just a Bunch of Drives 感觉和 raid0 概念类似

<http://en.wikipedia.org/wiki/Non-RAID_drive_architectures>

megaraid 卡使用 JBOD 模式，磁盘可以直接被系统识别，使用 smartctl 查看 SMART 信息和 **直连 SAS** 卡一样。

如果 LSI megaraid 卡没有启用 JBOD 模式，磁盘必须做 raid 操作，才能被系统识别到 。

没有启动 JBOD 模式，没法使用 megacli 设置磁盘为 JBOD 。查看 JBOD 是否启用：

    # megacli -AdpGetProp -enablejbod -aALL

    Adapter 0: JBOD: Disabled

    Exit Code: 0x00

没有开启，先启用该特性：

    # megacli -h
    ... ...
    MegaCli -AdpSetProp -EnableJBOD -val -aN|-a0,1,2|-aALL

          val - 0=Disable JBOD mode.
                1= Enable JBOD mode.

    # megacli -AdpSetProp -EnableJBOD -1 -aALL

    Adapter 0: Set JBOD to Enable success.

    Exit Code: 0x00

先手动删掉 RAID0 :

    # for i in {0..11}
    > do
    > megacli -CfgLdDel -L$i -a0 -Nolog
    > done

    Adapter 0: Deleted Virtual Drive-0(target id-0)

    Exit Code: 0x00
    ... ...

删完 RAID0 后，磁盘状态会变成 `Unconfigured(good), Spun Up` ：

    # megacli -PDList -aALL -Nolog|grep '^Firm'
    Firmware state: Unconfigured(good), Spun Up
    Firmware state: Unconfigured(good), Spun Up
    Firmware state: Unconfigured(good), Spun Up
    Firmware state: Unconfigured(good), Spun Up
    Firmware state: Unconfigured(good), Spun Up
    Firmware state: Unconfigured(good), Spun Up
    Firmware state: Unconfigured(good), Spun Up
    Firmware state: Unconfigured(good), Spun Up
    Firmware state: Unconfigured(good), Spun Up
    Firmware state: Unconfigured(good), Spun Up
    Firmware state: Unconfigured(good), Spun Up
    Firmware state: Unconfigured(good), Spun Up
    Firmware state: Online, Spun Up
    Firmware state: Online, Spun Up

此时就可以把磁盘改成 JBOD 模式了：

    # for i in {1..11}
    > do
    > megacli -PDMakeJBOD -PhysDrv[32:${i}] -a0
    > done

    Adapter: 0: EnclId-32 SlotId-1 state changed to JBOD.

    Exit Code: 0x00
    ...

完成后，再查看一下磁盘状态，已经变成了 JBOD 模式：

    # megacli -PDList -aALL -Nolog|grep '^Firm'
    Firmware state: JBOD
    Firmware state: JBOD
    Firmware state: JBOD
    Firmware state: JBOD
    Firmware state: JBOD
    Firmware state: JBOD
    Firmware state: JBOD
    Firmware state: JBOD
    Firmware state: JBOD
    Firmware state: JBOD
    Firmware state: JBOD
    Firmware state: JBOD
    Firmware state: Online, Spun Up
    Firmware state: Online, Spun Up

之前要用 slot ID 才能查看 SMART 信息：

    # smartctl -i -d sat+megaraid,6 /dev/sda
    smartctl 5.42 2011-10-20 r3458 [x86_64-linux-2.6.32-279.23.1.mi5.el6.x86_64] (local build)
    Copyright (C) 2002-11 by Bruce Allen, http://smartmontools.sourceforge.net

    === START OF INFORMATION SECTION ===
    Device Model:     INTEL SSDSC2BB240G4
    Serial Number:    BTWL3332042L240NGN
    LU WWN Device Id: 5 5cd2e4 04b4313be
    Firmware Version: D2010355
    User Capacity:    240,057,409,536 bytes [240 GB]
    Sector Size:      512 bytes logical/physical
    Device is:        Not in smartctl database [for details use: -P showall]
    ATA Version is:   8
    ATA Standard is:  ATA-8-ACS revision 4
    Local Time is:    Thu Jan 15 11:50:22 2014 CST
    SMART support is: Available - device has SMART capability.
    SMART support is: Enabled

现在直接通过盘符即可识别到磁盘：

    # smartctl -i /dev/sdm
    smartctl 5.42 2011-10-20 r3458 [x86_64-linux-2.6.32-279.23.1.mi5.el6.x86_64] (local build)
    Copyright (C) 2002-11 by Bruce Allen, http://smartmontools.sourceforge.net

    === START OF INFORMATION SECTION ===
    Device Model:     INTEL SSDSC2BB240G4
    Serial Number:    BTWL3332042L240NGN
    LU WWN Device Id: 5 5cd2e4 04b4313be
    Firmware Version: D2010355
    User Capacity:    240,057,409,536 bytes [240 GB]
    Sector Size:      512 bytes logical/physical
    Device is:        Not in smartctl database [for details use: -P showall]
    ATA Version is:   8
    ATA Standard is:  ATA-8-ACS revision 4
    Local Time is:    Thu Jan 15 11:56:27 2014 CST
    SMART support is: Available - device has SMART capability.
    SMART support is: Enabled









<br/>

本文标题 | [{{ page.title }}]({{ page.url }})
-------- |:--------
原始链接 | <{{ site.url }}{{ page.url }}>
