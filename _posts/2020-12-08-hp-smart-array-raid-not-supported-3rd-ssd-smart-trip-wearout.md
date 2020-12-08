---
layout: post
title: "HP RAID 卡无法识别第三方 SSD 的 Wearout 状态"
category: hardware
tags: [RAID]
---

# WHAT

HP RAID 卡无法获取 Intel 960G SSD 读写寿命 Wearout 状态：`SSD Smart Trip Wearout: Not Supported`

    # ssacli ctrl slot=0 pd 1I:1:3 show detail

    Smart Array P440ar in Slot 0 (Embedded)

       Array B

          physicaldrive 1I:1:3
             Port: 1I
             Box: 1
             Bay: 3
             Status: OK
             Drive Type: Data Drive
             Interface Type: Solid State SATA
             Size: 960 GB
             Drive exposed to OS: False
             Logical/Physical Block Size: 512/4096
             Firmware Revision: XCV10100
             Serial Number: PHYF0002960CGN
             WWID: 3001438040309B
             Model: ATA     INTEL SSDSC2KB96
             SATA NCQ Capable: True
             SATA NCQ Enabled: True
             Current Temperature (C): 20
             Maximum Temperature (C): 34
             SSD Smart Trip Wearout: Not Supported      <--
             PHY Count: 1
             PHY Transfer Rate: 6.0Gbps
             Drive Authentication Status: OK
             Carrier Application Version: 11
             Carrier Bootloader Version: 6
             Sanitize Erase Supported: True
             Unrestricted Sanitize Supported: True
             Shingled Magnetic Recording Support: None

# HOW

那就使用 `smartctl` 命令获取 Intel 960G SSD 硬盘的读写寿命信息：

> `man smartctl`
>
> `cciss,N` - [FreeBSD and Linux only] the device consists of one or more SCSI/SAS or SATA disks connected to a cciss RAID controller.
> The non-negative integer `N` (in the range from `0` to `15` inclusive) denotes which disk on the controller is monitored.

**注意**：`smartctl` 对应的 ID 是从 `0` 开始而 `ssacli` 工具对应的 `port:box:bay` ID 是从 `1` 开始

    # for i in {0..2}; do smartctl -i /dev/sda -d cciss,${i}|grep -i Serial; done
    Serial number:    S4220000K705MJRS
    Serial number:    S4220000K705MKBL
    Serial Number:    PHYF0002960CGN

    # ssacli ctrl slot=0 pd all show detail|grep -i serial
             Serial Number: S4220000K705MJRS
             Serial Number: S4220000K705MKBL
             Serial Number: PHYF0002960CGN

使用 `smartctl` 命令获取 `Media_Wearout_Indicator` 状态：

    # smartctl -A /dev/sda -d sat+cciss,2 -l ssd
    smartctl 6.2 2013-07-26 r3841 [x86_64-linux-3.10.0-327.el7.x86_64] (local build)
    Copyright (C) 2002-13, Bruce Allen, Christian Franke, www.smartmontools.org

    === START OF READ SMART DATA SECTION ===
    SMART Attributes Data Structure revision number: 1
    Vendor Specific SMART Attributes with Thresholds:
    ID# ATTRIBUTE_NAME          FLAG     VALUE WORST THRESH TYPE      UPDATED WHEN_FAILED RAW_VALUE
      5 Reallocated_Sector_Ct   0x0032   100   100   000    Old_age   Always      -       0
      9 Power_On_Hours          0x0032   100   100   000    Old_age   Always      -       6384
     12 Power_Cycle_Count       0x0032   100   100   000    Old_age   Always      -       7
    170 Unknown_Attribute       0x0033   100   100   010    Pre-fail  Always      -       0
    171 Unknown_Attribute       0x0032   100   100   000    Old_age   Always      -       0
    172 Unknown_Attribute       0x0032   100   100   000    Old_age   Always      -       0
    174 Unknown_Attribute       0x0032   100   100   000    Old_age   Always      -       7
    175 Program_Fail_Count_Chip 0x0033   100   100   010    Pre-fail  Always      -       34359675422
    183 Runtime_Bad_Block       0x0032   100   100   000    Old_age   Always      -       0
    184 End-to-End_Error        0x0033   100   100   090    Pre-fail  Always      -       0
    187 Reported_Uncorrect      0x0032   100   100   000    Old_age   Always      -       0
    190 Airflow_Temperature_Cel 0x0022   080   077   000    Old_age   Always      -       20 (Min/Max 14/23)
    192 Power-Off_Retract_Count 0x0032   100   100   000    Old_age   Always      -       7
    194 Temperature_Celsius     0x0022   100   100   000    Old_age   Always      -       20
    197 Current_Pending_Sector  0x0012   100   100   000    Old_age   Always      -       0
    199 UDMA_CRC_Error_Count    0x003e   100   100   000    Old_age   Always      -       0
    225 Unknown_SSD_Attribute   0x0032   100   100   000    Old_age   Always      -       492151
    226 Unknown_SSD_Attribute   0x0032   100   100   000    Old_age   Always      -       163
    227 Unknown_SSD_Attribute   0x0032   100   100   000    Old_age   Always      -       0
    228 Power-off_Retract_Count 0x0032   100   100   000    Old_age   Always      -       383023
    232 Available_Reservd_Space 0x0033   100   100   010    Pre-fail  Always      -       0
    233 Media_Wearout_Indicator 0x0032   100   100   000    Old_age   Always      -       0          <--
    234 Unknown_Attribute       0x0032   100   100   000    Old_age   Always      -       0
    235 Unknown_Attribute       0x0033   100   100   010    Pre-fail  Always      -       34359675422
    241 Total_LBAs_Written      0x0032   100   100   000    Old_age   Always      -       492151
    242 Total_LBAs_Read         0x0032   100   100   000    Old_age   Always      -       3
    243 Unknown_Attribute       0x0032   100   100   000    Old_age   Always      -       608986

    Device Statistics (GP Log 0x04)
    Page Offset Size         Value  Description
      7  =====  =                =  == Solid State Device Statistics (rev 1) ==
      7  0x008  1                0  Percentage Used Endurance Indicator

Intel 800G SSD `ssacli` 能获取 SSD 读写寿命，但是 `smartctl` 没有找到 `Media_Wearout_Indicator` 字段

`smartctl` 命令的 `-l ssd` 参数也可以获取 SSD 的使用百分比，参数 `-l` 的 man 手册：

> `man smartctl`
>
> `devstat[,PAGE]` - **[ATA only]** prints values and descriptions of the ATA Device Statistics log pages (General Purpose Log address 0x04).
> If no PAGE number is specified, entries from all supported pages are printed. If PAGE `0` is specified, the list of supported pages is printed.
> Device Statistics was introduced in ACS-2 and is only supported by some recent devices.
>
> `ssd` - [ATA] prints the Solid State Device Statistics log page. This has the same effect as `-l devstat,7`, see above.
>
> `ssd`  - [SCSI] prints the Solid State Media percentage used endurance indicator. A value of `0` indicates as new condition while `100` indicates the device is at the end of its lifetime as projected by the manufacturer. The value may reach `255`.

    # smartctl -A /dev/sda -d cciss,2 -l devstat,0
    smartctl 6.2 2013-07-26 r3841 [x86_64-linux-3.10.0-327.10.1.el7.x86_64] (local build)
    Copyright (C) 2002-13, Bruce Allen, Christian Franke, www.smartmontools.org

    /dev/sda [cciss_disk_02] [SAT]: Device open changed type from 'sat,auto+cciss' to 'sat'
    === START OF READ SMART DATA SECTION ===
    SMART Attributes Data Structure revision number: 1
    Vendor Specific SMART Attributes with Thresholds:
    ID# ATTRIBUTE_NAME          FLAG     VALUE WORST THRESH TYPE      UPDATED  WHEN_FAILED RAW_VALUE
      1 Raw_Read_Error_Rate     0x000f   130   130   039    Pre-fail  Always       -       2150939543
      5 Reallocated_Sector_Ct   0x0033   096   096   001    Pre-fail  Always       -       1
      9 Power_On_Hours          0x0032   100   100   000    Old_age   Always       -       43648
    173 Unknown_Attribute       0x0033   013   013   001    Pre-fail  Always       -       0
    175 Program_Fail_Count_Chip 0x0033   100   100   010    Pre-fail  Always       -       1112929469050
    180 Unused_Rsvd_Blk_Cnt_Tot 0x003b   130   130   039    Pre-fail  Always       -       4294967295
    194 Temperature_Celsius     0x0022   100   100   000    Old_age   Always       -       25
    196 Reallocated_Event_Count 0x0033   099   099   010    Pre-fail  Always       -       1

    Device Statistics (GP Log 0x04) supported pages
    Page Description
      0  List of supported log pages
      1  General Statistics
      4  General Errors Statistics
      5  Temperature Statistics
      6  Transport Statistics
      7  Solid State Device Statistics

    # smartctl -A /dev/sda -d cciss,2 -l ssd
    smartctl 6.2 2013-07-26 r3841 [x86_64-linux-3.10.0-327.10.1.el7.x86_64] (local build)
    Copyright (C) 2002-13, Bruce Allen, Christian Franke, www.smartmontools.org

    /dev/sda [cciss_disk_02] [SAT]: Device open changed type from 'sat,auto+cciss' to 'sat'
    === START OF READ SMART DATA SECTION ===
    SMART Attributes Data Structure revision number: 1
    Vendor Specific SMART Attributes with Thresholds:
    ID# ATTRIBUTE_NAME          FLAG     VALUE WORST THRESH TYPE      UPDATED  WHEN_FAILED RAW_VALUE
      1 Raw_Read_Error_Rate     0x000f   130   130   039    Pre-fail  Always       -       2150939543
      5 Reallocated_Sector_Ct   0x0033   096   096   001    Pre-fail  Always       -       1
      9 Power_On_Hours          0x0032   100   100   000    Old_age   Always       -       43648
    173 Unknown_Attribute       0x0033   013   013   001    Pre-fail  Always       -       0
    175 Program_Fail_Count_Chip 0x0033   100   100   010    Pre-fail  Always       -       1112929403514
    180 Unused_Rsvd_Blk_Cnt_Tot 0x003b   130   130   039    Pre-fail  Always       -       4294967295
    194 Temperature_Celsius     0x0022   100   100   000    Old_age   Always       -       24
    196 Reallocated_Event_Count 0x0033   099   099   010    Pre-fail  Always       -       1

    Device Statistics (GP Log 0x04)
    Page Offset Size         Value  Description
      7  =====  =                =  == Solid State Device Statistics (rev 1) ==
      7  0x008  1               87  Percentage Used Endurance Indicator

    # ssacli ctrl slot=0 pd 1I:1:3 show

    Smart Array P440ar in Slot 0 (Embedded)

       Array B

          physicaldrive 1I:1:3
             Port: 1I
             Box: 1
             Bay: 3
             Status: OK
             Drive Type: Data Drive
             Interface Type: Solid State SATA
             Size: 800 GB
             Drive exposed to OS: False
             Logical/Physical Block Size: 512/512
             Firmware Revision: 4IWVHPG1
             Serial Number: PHWL101QK800RGN
             WWID: 3001438028E617
             Model: ATA     VK0800GDJYA
             SATA NCQ Capable: True
             SATA NCQ Enabled: True
             Current Temperature (C): 24
             Maximum Temperature (C): 43
             Usage remaining: 12.94%
             Power On Hours: 43646
             Estimated Life Remaining based on workload to date: 270 days
             SSD Smart Trip Wearout: False
             PHY Count: 1
             PHY Transfer Rate: 6.0Gbps
             Drive Authentication Status: OK
             Carrier Application Version: 11
             Carrier Bootloader Version: 6
             Sanitize Erase Supported: False
             Shingled Magnetic Recording Support: None

    # ssacli ctrl slot=0 pd all show detail|grep -i 'remaining:'
             Usage remaining: 12.94%
             Usage remaining: -49.95%
             Usage remaining: 43.90%
             Usage remaining: 99.20%

    # for i in {2..5}; do smartctl -i /dev/sda -d cciss,${i} -l ssd|grep -i Percentage; done
      7  0x008  1               87  Percentage Used Endurance Indicator
      7  0x008  1              255  Percentage Used Endurance Indicator
      7  0x008  1               56  Percentage Used Endurance Indicator
      7  0x008  1                0  Percentage Used Endurance Indicator

扫描了一下其他机器的 SSD 读写使用寿命的关系：

`ssacli` | `smartctl`
-------- | ---------
`Usage remaining: 77.90%`    |  `7  0x008  1               22  Percentage Used Endurance Indicator`
`Usage remaining: 81.44% `   |  `7  0x008  1               18  Percentage Used Endurance Indicator`
`Usage remaining: -64.38% `  |  `7  0x008  1              255  Percentage Used Endurance Indicator`
`Usage remaining: -140.93% ` |  `7  0x008  1              240  Percentage Used Endurance Indicator`
`Usage remaining: 98.87%`    |  `7  0x008  1                1  Percentage Used Endurance Indicator`
`Usage remaining: 12.94%`    |  `7  0x008  1               87  Percentage Used Endurance Indicator`
`Usage remaining: 43.90%`    |  `7  0x008  1               56  Percentage Used Endurance Indicator`
`Usage remaining: 63.47%`    |  `7  0x008  1               36  Percentage Used Endurance Indicator`
`Usage remaining: 75.40%`    |  `7  0x008  1               24  Percentage Used Endurance Indicator`
`Usage remaining: -20.86%`   |  `7  0x008  1              120  Percentage Used Endurance Indicator`

# reference

<https://serverfault.com/questions/532405/3rd-party-ssd-drives-in-hp-proliant-server-monitoring-drive-health>

