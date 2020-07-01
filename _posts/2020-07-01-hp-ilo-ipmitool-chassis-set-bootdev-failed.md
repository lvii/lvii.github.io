---
layout: post
title: "HP 服务器无法使用 ipmitool 设置 PXE 引导选项"
category: system
tags: [IPMI]
---

# WHAT

厂里一台 HP 机器通过 `freeipmi` 和 `ipmitool` 设置 PXE 引导时提示失败报错：

    # ipmi-chassis-config -D LAN_2_0 -u root -p xxx -h 10.23.10.29 -c -e Chassis_Boot_Flags:Boot_Device=PXE
    ERROR: Failed to commit `Chassis_Boot_Flags:Boot_Device'

    # ipmitool -I lanplus -U root -f ~/.oob -H 10.23.10.29 chassis bootdev pxe
    Set Chassis Boot Parameter 5 failed: Command response could not be provided

通过 `freeipmi` 工具排除 IPMI 用户设置问题：

    # ipmipower -D LAN_2_0 -u root -p xxx -h 10.23.10.29 --on-if-off --stat
    10.23.10.29: on

查看电源状态，读取配置等操作都是正常的：

    # ipmi-chassis-config -D LAN_2_0 -u root -p xxx -h 10.23.10.29 -o -e Chassis_Boot_Flags:Boot_Device
    Section Chassis_Boot_Flags
            ## Possible values: NO-OVERRIDE/PXE/HARD-DRIVE/HARD-DRIVE-SAFE-MODE/
            ##                  DIAGNOSTIC_PARTITION/CD-DVD/BIOS-SETUP/REMOTE-FLOPPY
            ##                  PRIMARY-REMOTE-MEDIA/REMOTE-CD-DVD/REMOTE-HARD-DRIVE/FLOPPY
    Boot_Device                                   HARD-DRIVE
    EndSection

尝试重启 iLo 管理卡，依然无效。

# HOW

于是手动重启，尝试 `F12` 进入 PXE 引导，重启出现下面的提示：

![img](https://i.imgur.com/tSMfPm8.png)

原来是配置重置，导致配置项 **只读**。

重新初始化后，再次使用 `ipmitool` 或 `freeipmi` 可以正常修改引导选项了。

<br/>

本文标题 | [{{ page.title }}]({{ page.url }})
-------- |:--------
原始链接 | <{{ site.url }}{{ page.url }}>