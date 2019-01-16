---
layout: post
title: "VMware 扩容 Fedora 虚拟机磁盘后调整分区并拉伸文件系统"
category: system
tags: [vmware, fedora]
date: 2018-10-10 11:00:00 +08:00
---

Fedora 29 前两天已经 Final Freeze 了，准备从 Fedora 28 升级到 Fedora 29 。
之前创建虚拟机时，硬盘设置的有点小 upgrade 这种操作空间不够用，在 VMware 中把虚机硬盘扩容至 `8 GiB` 。
虚拟机 **只有一个 `/` 分区**，后面调整磁盘分区，拉伸文件系统，需要在 LiveOS 环境进行操作，
下载了一个 netinst ISO 安装镜像，重启引导进入安装 LiveOS 环境。

## 调整分区

查看当前分区的 **起始位置** 和 **分区标记 ( Flags )** ：

    [anaconda root@localhost /]# parted /dev/sda
    GNU Parted 3.2
    Using /dev/sda
    Welcome to GNU Parted! Type 'help' to view a list of commands.
    (parted) p
    Model: VMware, VMware Virtual S (scsi)
    Disk /dev/sda: 8590MB
    Sector size (logical/physical): 512B/512B
    Partition Table: msdos
    Disk Flags:

    Number  Start   End     Size    Type     File system  Flags
     1      1049kB  5369MB  5368MB  primary  ext4         boot

删除旧分区：

    (parted) rm 1

    (parted) p
    Model: VMware, VMware Virtual S (scsi)
    Disk /dev/sda: 8590MB
    Sector size (logical/physical): 512B/512B
    Partition Table: msdos
    Disk Flags:

    Number  Start  End  Size  Type  File system  Flags

从原分区的 **起始位置** 开始新建 **主分区** 至 **最大可用空间** ：

    (parted) mkpart primary 1049kB 100%

    (parted) p
    Model: VMware, VMware Virtual S (scsi)
    Disk /dev/sda: 8590MB
    Sector size (logical/physical): 512B/512B
    Partition Table: msdos
    Disk Flags:

    Number  Start   End     Size    Type     File system  Flags
     1      1049kB  8590MB  8589MB  primary               lba

重新创建分区后，之前的 **分区标记** `Flags` 丢失，重新设置 `boot` 不然无法启动：

    (parted) set 1 boot on

    (parted) p
    Model: VMware, VMware Virtual S (scsi)
    Disk /dev/sda: 8590MB
    Sector size (logical/physical): 512B/512B
    Partition Table: msdos
    Disk Flags:

    Number  Start   End     Size    Type     File system  Flags
     1      1049kB  8590MB  8589MB  primary               boot, lba

    (parted) set 1 lba off

    (parted) p
    Model: VMware, VMware Virtual S (scsi)
    Disk /dev/sda: 8590MB
    Sector size (logical/physical): 512B/512B
    Partition Table: msdos
    Disk Flags:

    Number  Start   End     Size    Type     File system  Flags
     1      1049kB  8590MB  8589MB  primary               boot

    (parted) q
    Information: You may need to update /etc/fstab.

**TIPS：** 使用 `growpart` 可以更简单的扩展分区，树莓派就是用 `growpart` 扩展 SD 卡分区的：

<https://docs.fedoraproject.org/en-US/quick-docs/raspberry-pi/#booting-fedora-on-a-raspberry-pi-for-the-first-time_rpi>

<https://fedoraproject.org/wiki/Architectures/ARM/Raspberry_Pi#Resize_after_initial-setup>

## 文件系统自检

分区调整好后，使用 `e2fsck` **自检** 分区上的文件系统：

    [anaconda root@localhost /]# e2fsck -fv /dev/sda1
    e2fsck 1.44.3 (10-July-2018)
    Superblock last write time is in the future.
            (by less than a day, probably due to the hardware clock being incorrectly set)
    Pass 1: Checking inodes, blocks, and sizes
    Inode 138584 extent tree (at level 1) could be shorter.  Fix<y>? yes
    Inode 282934 extent tree (at level 1) could be shorter.  Fix<y>? yes
    Pass 1E: Optimizing extent trees
    Pass 2: Checking directory structure
    Pass 3: Checking directory connectivity
    Pass 4: Checking reference counts
    Pass 5: Checking group summary information

    fedora-root: ***** FILE SYSTEM WAS MODIFIED *****

          141558 inodes used (43.20%, out of 327680)
            2435 non-contiguous files (1.7%)
             149 non-contiguous directories (0.1%)
                 # of inodes with ind/dind/tind blocks: 0/0/0
                 Extent depth histogram: 127429/623/1
         1106492 blocks used (84.44%, out of 1310464)
               0 bad blocks
               1 large file

          111258 regular files
           15944 directories
               0 character device files
               0 block device files
               0 fifos
           11243 links
           14347 symbolic links (13497 fast symbolic links)
               0 sockets
    ------------
          152792 files

## 文件系统扩容

使用 `resize2fs` 对文件系统进行拉伸：

    [anaconda root@localhost /]# resize2fs /dev/sda1
    resize2fs 1.44.3 (10-July-2018)
    Resizing the filesystem on /dev/sda1 to 2096896 (4k) blocks.
    The filesystem on /dev/sda1 is now 2096896 (4k) blocks long.

使用 `dumpe2fs` 确认文件系统大小：[How to find the size of a filesystem ? ](https://askubuntu.com/questions/622489/how-to-find-the-size-of-a-filesystem/622523#622523)

    # echo 1024*1024*1024|bc
    1073741824

    # dumpe2fs -h /dev/sda1 |& awk -F: '/Block count/{count=$2} /Block size/{size=$2} END{print count*size/1073741824}'
    7.99902

也可以挂载 `/dev/sda1` 到 `/mnt` 查看大小：

    [anaconda root@localhost /]# mount /dev/sda1 /mnt

    [anaconda root@localhost /]# lsblk
    NAME        MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
    loop0         7:0    0  482M  1 loop
    loop1         7:1    0    2G  1 loop
    ├─live-rw   253:0    0    2G  0 dm   /
    └─live-base 253:1    0    2G  1 dm
    loop2         7:2    0   32G  0 loop
    └─live-rw   253:0    0    2G  0 dm   /
    sda           8:0    0    8G  0 disk
    └─sda1        8:1    0    8G  0 part /mnt
    sr0          11:0    1  594M  0 rom  /run/install/repo

<br/>

本文标题 | [{{ page.title }}]({{ page.url }})
-------- |:--------
原始链接 | <{{ site.url }}{{ page.url }}>
