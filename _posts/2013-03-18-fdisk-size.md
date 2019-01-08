---
layout: post
title: fdisk 引发的 disk size 探究
category: system
tags: [fdisk, parted, lvm]
---

最近遇到脚本中 `fdisk` 获取磁盘大小和 LVM 处理逻辑卷时，两者对磁盘分区大小计算不同

仔细捣腾了下，那些和磁盘分区有关的工具 size 问题

## fdisk

    # fdisk -l /dev/sda2

    Disk /dev/sda2: 15.4 GB, 15364823040 bytes
    255 heads, 63 sectors/track, 1868 cylinders
    Units = cylinders of 16065 * 512 = 8225280 bytes
    Sector size (logical/physical): 512 bytes / 512 bytes
    I/O size (minimum/optimal): 512 bytes / 512 bytes
    Disk identifier: 0x00000000

关于 fdisk 输出的几个 size 相关数字解析：

- 分区大小 byte ：`255 * 63 * 1818 * 512 = 15364823040` ( google 道听途说的)
- 分区大小 1000 进制：`15.4G`
- 分区大小 1024 进制：`15364823040.0 / 1073741824 = 14.3`

### 物理结构

`fdisk` 显示的 **字节大小** 为啥要 `heads * sectors * cylinders * block_size`

关于 heads sectors cylinders 名词解释，需要了解硬盘的 **物理结构** ：

![Cylinder_Head_Sector.svg](http://upload.wikimedia.org/wikipedia/commons/0/02/Cylinder_Head_Sector.svg)
![ZBR.jpb](http://www.msexchange.org/img/upl/image0031118243018869.jpg)

采用同心圆方式 **均分扇区** 还是古老的硬盘采用的方式。新式硬盘采用 ZBR (Zoned Bit Recording) 方式 **分组均分**

关于硬盘结构介绍可以看这篇公开课 [Disk Drive Terms and Concepts](http://www.c-jump.com/CIS24/Slides/DiskDrives/DiskDrives.html)
图文并茂，非常详细

科普完后 `fdisk` 应该仅仅是借用了硬盘结构的概念而已，哪会有 `255` 个 heads 啊 -_-#

### 寻址方式

那是否和 **寻址方式** 有关，应该是没关系

因为 `fdisk` 字节大小计算和 wikipedia 中理论的 LBA 寻址计算方式不一样:

根据 [wikipedia LBA 逻辑寻址计算](http://en.wikipedia.org/wiki/Logical_block_addressing) 是根据磁盘 heads sectors cylinders 的 **实际值**

`fdisk` 显示的 heads sectors cylinders 都是 **逻辑值**，而不是 **物理值**

参考这篇博文：[关于 fdisk -l 看到的 heads](http://zhumeng8337797.blog.163.com/blog/static/100768914201010183442986)

> heads 值 `255` ，heads 表示 **磁头数**，而普通硬盘的磁头数最多也就是 `4` 个  
> heads 原来是逻辑数值 `fdisk -l` 看到的 `sectors` 和 `cylinders` 也是 **逻辑值**  
> LBA 模式下设置的柱面、磁头、扇区等参数并不是实际硬盘的物理参数

看来要想知道为啥这样子算，要读源码了 ...

## parted

下面看看 parted 指定 `unit GiB` 获得的分区大小：

    # parted /dev/sda2 unit GiB print
    Model: Unknown (unknown)
    Disk /dev/sda2: 14.3GiB
    Sector size (logical/physical): 512B/512B
    Partition Table: loop

    Number  Start    End      Size     File system  Flags
     1      0.00GiB  14.3GiB  14.3GiB  ext4

`parted` 的 `unit` 参数可以指定使用 `1000` 进制 (默认) `GB` 或 `1024` 进制 `GiB` 来显示分区大小

其他可选参数还是看手册吧：[http://www.gnu.org/software/parted/manual/html_node/unit.html](http://www.gnu.org/software/parted/manual/html_node/unit.html)

    # parted /dev/sda2 unit GiB print devices
    /dev/sda2 (14.3GiB)
    /dev/sda (931GiB)

## LVM

像 `fdisk` 和 `parted` 对分区大小的计算默认都是以 `1000` 进制的来计算 size 的。
但是当要用到 lvm 这货相关的命令，刚好和 `fdisk` , `parted` 相反，默认使用 `1024` 进制的。
至少 lvm 相关的显示命令是可以通过 `--unit` 大小写字母来指定使用 `1000` 还是 `1024` 进制的：

    # pvs --noheadings -o pv_name,pv_size
      /dev/sda6  546.50G

    # pvs --noheadings --unit G -o pv_name,pv_size
      /dev/sda6  586.80G

但悲摧的是 `lvcreate` 的 `--size` 参数竟然和 `lvs` 命令不一样：

    man lvcreate
    ... ...
    -L, --size LogicalVolumeSize[bBsSkKmMgGtTpPeE]
        Gives the size to allocate for the new logical volume.
        A size suffix of K for kilobytes, M for megabytes, G for  gigabytes,
        T for terabytes, P for petabytes or E for exabytes is optional.
        Default unit is megabytes.

    # lvcreate -L 10G vg -n unit_G
      Logical volume "unit_G" created

    # lvcreate -L 10g vg -n unit_g
      Logical volume "unit_g" created

    # lvs
      LV     VG   Attr     LSize
      unit_G vg   -wi-a--- 10.00g
      unit_g vg   -wi-a--- 10.00g

`lvcreate` 的 `--size` 根本无视大小写，一视同仁使用 `1024` 进制，怎么觉得不像是一家人开发的 -_-;

so 当要使用 lvm 时，要把 `fdisk` 和 `parted` 对应的分区换算成 `1024` 进制的 size 避免出现空间不够的杯具 ...

## 参考

[Disk Geometry](http://www.msexchange.org/articles-tutorials/exchange-server-2003/planning-architecture/Disk-Geometry.html)




<br/>

本文标题 | [{{ page.title }}]({{ page.url }})
-------- |:--------
原始链接 | <{{ site.url }}{{ page.url }}>
