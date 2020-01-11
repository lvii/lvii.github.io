---
layout: post
title: "使用 AMI SCELNX 工具配置 BIOS"
category: hardware
tags: [BIOS]
---

# WHY

最近厂里采购的 H3C 机器默认 **PXE 网络引导** 为第一启动项，
奇葩的是 PXE 失败后不会继续使用第二启动项 **硬盘引导** ，
导致机器无法进入系统，只能通过 `ipmitool` 设置从硬盘引导，才能进入系统。
针对这问题只能修改 BIOS 默认为 **硬盘引导**

# WHAT


[AMI Setup Control Environment (AMISCE)](https://ami.com/en/products/bios-uefi-tools-and-utilities/bios-uefi-utilities/)

> **AMISCE** is a command line tool which provides an easy way to update NVRAM variables, extract variables directly from the BIOS, change settings using either a text editor or a setup program and update the BIOS. AMISCE produces a script file that lists all setup questions on the system being modified by AMISCE. The user can then modify the script file and use it as input to change the current NVRAM setup variables.

AMISCE 在 Linux 下通过 `SCELNX_64` 命令行工具来配置 BIOS

# HOW

`SCELNX_64` 依赖 `kernel-devel` 软件包：

    yum install -y kernel-devel

## NVRAM Script File

读取并保存 AMI BIOS 配置文件 **NVRAM Script File** ：

    # SCELNX_64 /o /lang /s BIOS-with-map-string.cfg /hb
    WARNING: Duplicate questions found. Only one question will be enabled in the script.
    Script file exported successfully.

上面命令提示有重复的 question 使用 `/sd` 参数将重复 question 另存为单独文件：

    # SCELNX_64 /o /lang /s BIOS-with-map-string.cfg /sd BIOS-duplicate-questions.cfg /hb
    WARNING: Duplicate questions found. Only one question will be enabled in the script.
    Script file exported successfully.

重复的 question 基本都是 **被注释** 的选项：

    # grep 'Boot mode select' *
    BIOS-with-map-string.cfg:Setup Question = Boot mode select
    BIOS-with-map-string.cfg:// Setup Question      = Boot mode select
    BIOS-duplicate-questions.cfg:// Setup Question  = Boot mode select
    BIOS-uniq-with-map-string.cfg:Setup Question    = Boot mode select

## map string

使用 `/lang` 参数导出的配置文件，才包含 BIOS 配置项对应的 map string ：

    Setup Question  = Boot Option #1
    Map String      = FBO201
    Token   =1080   // Do NOT change this line
    Offset  =FE
    Width   =02
    BIOS Default =[00]Hard Disk
    MFG Default =[00]Hard Disk
    Options =[00]Hard Disk  // Move "*" to the desired Option
             *[01]Network
             [02]USB Hard Disk
             [03]USB CD/DVD
             [04]USB Key
             [05]USB Floppy
             [06]CD/DVD
             [07]UEFI AP
             [08]Disabled

    Setup Question  = Boot Option #2
    Map String      = FBO202
    Token   =1081   // Do NOT change this line
    Offset  =100
    Width   =02
    BIOS Default =[01]Network
    MFG Default =[01]Network
    Options =*[00]Hard Disk // Move "*" to the desired Option
             [01]Network
             [02]USB Hard Disk
             [03]USB CD/DVD
             [04]USB Key
             [05]USB Floppy
             [06]CD/DVD
             [07]UEFI AP
             [08]Disabled

    Setup Question  = Boot Option #1
    Map String      = FBO101
    Token   =1000   // Do NOT change this line
    Offset  =DE
    Width   =02
    BIOS Default =[00]Hard Disk:ASR-8060-RAID RAID Ctlr #0
    MFG Default =[00]Hard Disk:ASR-8060-RAID RAID Ctlr #0
    Options =[00]Hard Disk:ASR-8060-RAID RAID Ctlr #0       // Move "*" to the desired Option
             *[01]Network:IBA XE Slot 5F00 v2346
             [02]USB Hard Disk
             [03]USB CD/DVD
             [04]USB Key:AMI Virtual HDisk0 1.00
             [05]USB Floppy
             [06]CD/DVD
             [07]Disabled

    Setup Question  = Boot Option #2
    Map String      = FBO102
    Token   =1001   // Do NOT change this line
    Offset  =E0
    Width   =02
    BIOS Default =[01]Network:IBA XE Slot 5F00 v2346
    MFG Default =[01]Network:IBA XE Slot 5F00 v2346
    Options =*[00]Hard Disk:ASR-8060-RAID RAID Ctlr #0      // Move "*" to the desired Option
             [01]Network:IBA XE Slot 5F00 v2346
             [02]USB Hard Disk
             [03]USB CD/DVD
             [04]USB Key:AMI Virtual HDisk0 1.00
             [05]USB Floppy
             [06]CD/DVD
             [07]Disabled

## config

查看 BIOS 单一配置选项的命令格式：

    Single Question Export Usage:
        SCELNX_64 /o [/lang <Lang Code>] /ms <question map string> [/q] [/d] [/hb]

根据 map string 查询单独配置项：

    # SCELNX_64 /o /ms FBO201 /hb
    Options =[00]Hard Disk
             *[01]Network
             [02]USB Hard Disk
             [03]USB CD/DVD
             [04]USB Key
             [05]USB Floppy
             [06]CD/DVD
             [07]UEFI AP
             [08]Disabled

根据 map string 修改单独配置项：

    # SCELNX_64 /i /ms FBO201 /qv 0x00 /hb

    Question value imported successfully.

    # SCELNX_64 /o /lang /ms FBO201 /hb
    Options =*[00]Hard Disk
             [01]Network
             [02]USB Hard Disk
             [03]USB CD/DVD
             [04]USB Key
             [05]USB Floppy
             [06]CD/DVD
             [07]UEFI AP
             [08]Disabled

## boot order config

**注意：** boot order 有 2 组配置，只修改第一组没用 。。。

    Setup Question  = Boot Option #1
    Map String      = FBO201
    Token   =1080   // Do NOT change this line
    Offset  =FE
    Width   =02
    BIOS Default =[00]Hard Disk
    MFG Default =[00]Hard Disk
    Options =[00]Hard Disk  // Move "*" to the desired Option
             *[01]Network
             [02]USB Hard Disk
             [03]USB CD/DVD
             [04]USB Key
             [05]USB Floppy
             [06]CD/DVD
             [07]UEFI AP
             [08]Disabled

    Setup Question  = Boot Option #2
    Map String      = FBO202
    Token   =1081   // Do NOT change this line
    Offset  =100
    Width   =02
    BIOS Default =[01]Network
    MFG Default =[01]Network
    Options =*[00]Hard Disk // Move "*" to the desired Option
             [01]Network
             [02]USB Hard Disk
             [03]USB CD/DVD
             [04]USB Key
             [05]USB Floppy
             [06]CD/DVD
             [07]UEFI AP
             [08]Disabled

    Setup Question  = Boot Option #1
    Map String      = FBO101
    Token   =1000   // Do NOT change this line
    Offset  =DE
    Width   =02
    BIOS Default =[00]Hard Disk:ASR-8060-RAID RAID Ctlr #0
    MFG Default =[00]Hard Disk:ASR-8060-RAID RAID Ctlr #0
    Options =[00]Hard Disk:ASR-8060-RAID RAID Ctlr #0       // Move "*" to the desired Option
             *[01]Network:IBA XE Slot 5F00 v2346
             [02]USB Hard Disk
             [03]USB CD/DVD
             [04]USB Key:AMI Virtual HDisk0 1.00
             [05]USB Floppy
             [06]CD/DVD
             [07]Disabled

    Setup Question  = Boot Option #2
    Map String      = FBO102
    Token   =1001   // Do NOT change this line
    Offset  =E0
    Width   =02
    BIOS Default =[01]Network:IBA XE Slot 5F00 v2346
    MFG Default =[01]Network:IBA XE Slot 5F00 v2346
    Options =*[00]Hard Disk:ASR-8060-RAID RAID Ctlr #0      // Move "*" to the desired Option
             [01]Network:IBA XE Slot 5F00 v2346
             [02]USB Hard Disk
             [03]USB CD/DVD
             [04]USB Key:AMI Virtual HDisk0 1.00
             [05]USB Floppy
             [06]CD/DVD
             [07]Disabled

修改两组 boot order 对应的 BIOS 配置项：

    SCELNX_64 /i /ms FBO101 /qv 0x00 /hb
    SCELNX_64 /i /ms FBO102 /qv 0x01 /hb
    SCELNX_64 /i /ms FBO201 /qv 0x00 /hb
    SCELNX_64 /i /ms FBO202 /qv 0x01 /hb

修改完后查看是否生效：

    # SCELNX_64 /o /ms FBO201 /hb
    Options =*[00]Hard Disk
             [01]Network
             [02]USB Hard Disk
             [03]USB CD/DVD
             [04]USB Key
             [05]USB Floppy
             [06]CD/DVD
             [07]UEFI AP
             [08]Disabled

    # SCELNX_64 /o /ms FBO202 /hb
    Options =[00]Hard Disk
             *[01]Network
             [02]USB Hard Disk
             [03]USB CD/DVD
             [04]USB Key
             [05]USB Floppy
             [06]CD/DVD
             [07]UEFI AP
             [08]Disabled

    # SCELNX_64 /o /ms FBO101 /hb
    Options =*[00]Hard Disk:ASR-8060-RAID RAID Ctlr #0
             [01]Network:IBA XE Slot 5F00 v2346
             [02]USB Hard Disk
             [03]USB CD/DVD
             [04]USB Key:AMI Virtual HDisk0 1.00
             [05]USB Floppy
             [06]CD/DVD
             [07]Disabled

    # SCELNX_64 /o /ms FBO102 /hb
    Options =[00]Hard Disk:ASR-8060-RAID RAID Ctlr #0
             *[01]Network:IBA XE Slot 5F00 v2346
             [02]USB Hard Disk
             [03]USB CD/DVD
             [04]USB Key:AMI Virtual HDisk0 1.00
             [05]USB Floppy
             [06]CD/DVD
             [07]Disabled

对应的 BIOS 配置文件：

    Setup Question  = Boot Option #1
    Map String      = FBO201
    Token   =1080   // Do NOT change this line
    Offset  =FE
    Width   =02
    BIOS Default =[00]Hard Disk
    MFG Default =[00]Hard Disk
    Options =*[00]Hard Disk // Move "*" to the desired Option
             [01]Network
             [02]USB Hard Disk
             [03]USB CD/DVD
             [04]USB Key
             [05]USB Floppy
             [06]CD/DVD
             [07]UEFI AP
             [08]Disabled

    Setup Question  = Boot Option #2
    Map String      = FBO202
    Token   =1081   // Do NOT change this line
    Offset  =100
    Width   =02
    BIOS Default =[01]Network
    MFG Default =[01]Network
    Options =[00]Hard Disk  // Move "*" to the desired Option
             *[01]Network
             [02]USB Hard Disk
             [03]USB CD/DVD
             [04]USB Key
             [05]USB Floppy
             [06]CD/DVD
             [07]UEFI AP
             [08]Disabled

    Setup Question  = Boot Option #1
    Map String      = FBO101
    Token   =1000   // Do NOT change this line
    Offset  =DE
    Width   =02
    BIOS Default =[00]Hard Disk:ASR-8060-RAID RAID Ctlr #0
    MFG Default =[00]Hard Disk:ASR-8060-RAID RAID Ctlr #0
    Options =*[00]Hard Disk:ASR-8060-RAID RAID Ctlr #0      // Move "*" to the desired Option
             [01]Network:IBA XE Slot 5F00 v2346
             [02]USB Hard Disk
             [03]USB CD/DVD
             [04]USB Key:AMI Virtual HDisk0 1.00
             [05]USB Floppy
             [06]CD/DVD
             [07]Disabled

    Setup Question  = Boot Option #2
    Map String      = FBO102
    Token   =1001   // Do NOT change this line
    Offset  =E0
    Width   =02
    BIOS Default =[01]Network:IBA XE Slot 5F00 v2346
    MFG Default =[01]Network:IBA XE Slot 5F00 v2346
    Options =[00]Hard Disk:ASR-8060-RAID RAID Ctlr #0       // Move "*" to the desired Option
             *[01]Network:IBA XE Slot 5F00 v2346
             [02]USB Hard Disk
             [03]USB CD/DVD
             [04]USB Key:AMI Virtual HDisk0 1.00
             [05]USB Floppy
             [06]CD/DVD
             [07]Disabled

然后重启服务器确认修改是否生效

<br/>

本文标题 | [{{ page.title }}]({{ page.url }})
-------- |:--------
原始链接 | <{{ site.url }}{{ page.url }}>
