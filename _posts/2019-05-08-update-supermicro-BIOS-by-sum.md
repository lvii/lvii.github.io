---
layout: post
title: "使用 SUM 工具修改超微 BIOS 配置"
category: system
tags: [bios]
---

# WHY

厂里有一批 **过保** 的 Supermicro 3U8 机器 CPU 没有启用 **超线程**

虽然超微 3U8 机型 [X9SRD-F](https://www.supermicro.com/products/motherboard/Xeon/C600/X9SRD-F.cfm) 主板使用的也是 AMI 的 BIOS

> BIOS Type: 32Mb SPI Flash EEPROM with ***AMI BIOS***

之前针对 H3C 机型的 `SCELNX` 工具不能用：

    # SCELNX_64 /o /lang /s BIOS-with-map-string.cfg /hb
    Platform identification failed.

看来得用超微自己的 SUM 工具来修改

# WHAT

[Supermicro Update Manager (SUM)](https://www.supermicro.com/solutions/SMS_SUM.cfm)

> BIOS Management
> - Update BIOS Firmware
> - BIOS Information
> - Get Current/Default BIOS Settings
> - Change BIOS settings
> - Get/Change/Edit DMI Information

![img](https://i.imgur.com/mIjp5kF.png)

下载文件里面有 `sum` 命令的详细说明文档：`SUM_UserGuide.pdf`

    # tar tf sum_2.2.0_Linux_x86_64_20190220.tar.gz
    sum_2.2.0_Linux_x86_64/
    sum_2.2.0_Linux_x86_64/ExternalData/
    sum_2.2.0_Linux_x86_64/ExternalData/SMCIPID.txt
    sum_2.2.0_Linux_x86_64/ExternalData/VENID.txt
    sum_2.2.0_Linux_x86_64/ExternalData/tui.fnt
    sum_2.2.0_Linux_x86_64/sumrc.sample
    sum_2.2.0_Linux_x86_64/SUM_UserGuide.pdf                <--
    sum_2.2.0_Linux_x86_64/driver/
    sum_2.2.0_Linux_x86_64/driver/RHL7_x86_64/
    sum_2.2.0_Linux_x86_64/driver/RHL7_x86_64/sum_bios.ko
    sum_2.2.0_Linux_x86_64/driver/RHL6_x86_64/
    sum_2.2.0_Linux_x86_64/driver/RHL6_x86_64/sum_bios.ko
    sum_2.2.0_Linux_x86_64/driver/RHL4_x86_64/
    sum_2.2.0_Linux_x86_64/driver/RHL4_x86_64/sum_bios.ko
    sum_2.2.0_Linux_x86_64/driver/RHL5_x86_64/
    sum_2.2.0_Linux_x86_64/driver/RHL5_x86_64/sum_bios.ko
    sum_2.2.0_Linux_x86_64/sum
    sum_2.2.0_Linux_x86_64/ReleaseNote.txt

# HOW

当前 CPU 没启用超线程只有 4 个核：

    # dmidecode -t processor | grep -E '(Core Count|Thread Count)'
            Core Count: 4
            Thread Count: 8

    # lscpu
    Architecture:          x86_64
    CPU op-mode(s):        32-bit, 64-bit
    Byte Order:            Little Endian
    CPU(s):                4                    <--
    On-line CPU(s) list:   0-3
    Thread(s) per core:    1                    <--
    Core(s) per socket:    4
    Socket(s):             1
    NUMA node(s):          1
    Vendor ID:             GenuineIntel
    CPU family:            6
    Model:                 62
    Model name:            Intel(R) Xeon(R) CPU E5-1620 v2 @ 3.70GHz
    Stepping:              4
    CPU MHz:               3701.000
    BogoMIPS:              7400.40
    Virtualization:        VT-x
    L1d cache:             32K
    L1i cache:             32K
    L2 cache:              256K
    L3 cache:              10240K
    NUMA node0 CPU(s):     0-3

## License Key

使用 `sum` 工具修改 BIOS 需要 License Key 厂里的机器也都没有 `-_-;`

好在超微 **旧机型(2012-2018)** 的 License 算法已经被 **逆向** 出来了：

[Reverse Engineering Supermicro IPMI 2018-05-27](https://peterkleissner.com/2018/05/27/reverse-engineering-supermicro-ipmi/)

根据 OOB Mac 地址计算 License Key ：

    oob_mac_str=$(ipmitool lan print|awk '/^MAC Add/{print $4}')
    oob_hex_key=8544E3B47ECA58F9583043F8

    echo "$oob_mac_str"|xxd -r -p|openssl dgst -sha1 -mac HMAC -macopt "hexkey:${oob_hex_key}"|awk '{print $2}'|awk -v FIELDWIDTHS='4 4 4 4 4 4' -v OFS='-' '{$1=$1;print }'
    a2a5-e0de-ebbe-0862-53e2-5e16

导入 License Key ：

    # ./sum -c  ActivateProductKey --key a2a5-e0de-ebbe-0862-53e2-5e16
    Supermicro Update Manager (for UEFI BIOS) 2.2.0 (2019/02/20) (x86_64)
    Copyright(C)2019 Super Micro Computer, Inc. All rights reserved.
    Node product key (OOB) is activated for localhost.

    # ./sum -c QueryProductKey
    Supermicro Update Manager (for UEFI BIOS) 2.2.0 (2019/02/20) (x86_64)
    Copyright(C)2019 Super Micro Computer, Inc. All rights reserved.

    [0] OOB
    Number of product keys: 1

然后就可以使用 `sum` 工具配置 BIOS 了，先查看当前 BIOS 里超线程 HT 的配置项：

    # ./sum -c GetCurrentBiosCfg |grep -C3 -i thread
    [Advanced|CPU Configuration]
    Clock Spread Spectrum=00                // *00 (Disabled), 01 (Enabled)
    RTID=00                                 // *00 (Optimal), 01 (Alternate)
    Hyper-threading=00                      // 00 (Disabled), *01 (Enabled)
    Limit CPUID Maximum=00                  // *00 (Disabled), 01 (Enabled)
    Execute Disable Bit=01                  // 00 (Disabled), *01 (Enabled)
    Intel(R) AES-NI=01                      // 00 (Disabled), *01 (Enabled)

当前没有启用，生成启用 HT 的 BIOS 配置文件：

    # echo '[Advanced|CPU Configuration]
    Hyper-threading=01                      // 00 (Disabled), *01 (Enabled)' > ht.cfg

    # cat ht.cfg
    [Advanced|CPU Configuration]
    Hyper-threading=01                      // 00 (Disabled), *01 (Enabled)

最后 `sum` 通过指定的 **配置文件** 修改 BIOS ：

    # ./sum -c ChangeBiosCfg --file ht.cfg
    Supermicro Update Manager (for UEFI BIOS) 2.2.0 (2019/02/20) (x86_64)
    Copyright(C)2019 Super Micro Computer, Inc. All rights reserved.

    Status: Start updating the BIOS configuration for the managed system

    ************************************WARNING*************************************
        Do not remove AC power from the server.
    ********************************************************************************

    Status: The BIOS configuration is updated for the managed system

    Note: You have to reboot or power up the system for the changes to take effect

修改成功后，还需 **重启** 服务器才能生效。重启之后，确认 HT 是否生效：

    # lscpu
    Architecture:          x86_64
    CPU op-mode(s):        32-bit, 64-bit
    Byte Order:            Little Endian
    CPU(s):                8                    <--
    On-line CPU(s) list:   0-7
    Thread(s) per core:    2                    <--
    Core(s) per socket:    4
    Socket(s):             1
    NUMA node(s):          1
    Vendor ID:             GenuineIntel
    CPU family:            6
    Model:                 62
    Model name:            Intel(R) Xeon(R) CPU E5-1620 v2 @ 3.70GHz
    Stepping:              4
    CPU MHz:               3701.000
    BogoMIPS:              7400.00
    Virtualization:        VT-x
    L1d cache:             32K
    L1i cache:             32K
    L2 cache:              256K
    L3 cache:              10240K
    NUMA node0 CPU(s):     0-7

    # ./sum -c GetCurrentBiosCfg |grep -C3 -i thread
    [Advanced|CPU Configuration]
    Clock Spread Spectrum=00                // *00 (Disabled), 01 (Enabled)
    RTID=00                                 // *00 (Optimal), 01 (Alternate)
    Hyper-threading=01                      // 00 (Disabled), *01 (Enabled)
    Limit CPUID Maximum=00                  // *00 (Disabled), 01 (Enabled)
    Execute Disable Bit=01                  // 00 (Disabled), *01 (Enabled)
    Intel(R) AES-NI=01                      // 00 (Disabled), *01 (Enabled)

