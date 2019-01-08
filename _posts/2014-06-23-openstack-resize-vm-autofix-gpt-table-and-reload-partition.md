---
layout: post
title: "openstack VM 磁盘扩容，修复 GPT 分区，更新分区表后，拉伸文件系统"
category: hardware
tags: [openwrt, route]
---

## 起因

之前，厂里 openstack 虚拟机，「云主机类型」不同模板定义的磁盘大小。是在原镜像的 「根磁盘」 上 **附加 (attach)** 相应大小的  「临时磁盘」 来实现的。在虚拟机里面看到的是 2 块独立的磁盘设备。将虚拟机「临时磁盘」扩容到逻辑卷组，以此做的快照启动，找不到之前附加的「临时磁盘」，杯具发生了： **起不来** `-_-!`

现在，不同模板直接指定不同大小的 「根磁盘」 ，不再使用 「临时磁盘」 。生成的虚拟机，就只看到一个 **扩容后 (resize)** 的磁盘。但由于母盘镜像大小和模板定义的磁盘大小不一样，parted 分区操作时，需要先修复分区，系统才能识别到相对母盘 **扩容** 的磁盘空间，然后分区，扩容逻辑卷组，拉伸文件系统，这样做的快照才可以正常使用。

## 经过

### 修复 GPT 分区

    # parted /dev/vda
    Using /dev/vda
    Welcome to GNU Parted! Type 'help' to view a list of commands.
    (parted) p
    Error: The backup GPT table is not at the end of the disk, as it should be.
    This might mean that another operating system believes the disk is smaller.
    Fix, by moving the backup to the end (and removing the old backup)?
    Fix/Ignore/Cancel? f
    Warning: Not all of the space available to /dev/vda appears to be used,
    you can fix the GPT to use all of the space (an extra 125829120 blocks) or continue
    with the current setting?
    Fix/Ignore? f

parted 没有可以自动 Fix 分区表的命令行参数，只好用 expect 交互式修复 GPT 分区，并扩增分区

    expect -c 'spawn parted /dev/vda; \
    expect "(parted)"; send -- "p\r"; \
    expect "Fix/Ignore/Cancel?"; send -- "f\r"; \
    expect "Fix/Ignore?"; send -- "f\r"; \
    expect "(parted)"; send -- "mkpart primary 42.9GB 100%\r"; \
    expect "(parted)"; send -- "p\r"; \
    expect "(parted)"; send -- "q\r"; \
    exit'

### 更新分区表

上面修复完 GPT 分区后，会出现下面的提示，要重启才能重新识别 **新分区** ：

    WARNING: the kernel failed to re-read the partition table on /dev/vda (Device or resource busy).
    As a result, it may not reflect all of your changes until after reboot.

parted 虽然可以看到新分区，但系统里找不到 **新分区** 对应的 block 设备

    # parted /dev/vda print
    Model: Virtio Block Device (virtblk)
    Disk /dev/vda: 129GB
    Sector size (logical/physical): 512B/512B
    Partition Table: gpt

    Number  Start   End     Size    File system  Name     Flags
     1      1049kB  211MB   210MB   ext4                  boot
     2      211MB   42.9GB  42.7GB                        lvm
     3      42.9GB  129GB   85.9GB               primary

    # ls /dev/vda
    vda  vda1  vda2

    # cat /proc/partitions
    major minor  #blocks  name

     252        0  125829120 vda
     252        1     204800 vda1
     252        2   41736192 vda2

尝试使用 `partprobe` 刷新分区表，结果无效。依然会提示 **reboot** 才能更新分区表

[\[SOLVED\] RHEL 6 "after partitioning the kernel didn't update the new partition it need reboot"](http://www.linuxquestions.org/questions/linux-server-73/rhel-6-after-partitioning-the-kernel-didn%27t-update-the-new-partition-it-need-reboot-916084/#post4540046)

> When you update the partition table, you need to tell the kernel it has been updated.
> Normally you can use `partprobe`.  However, on **RHEL6** , this no longer works for disks
> that have partitions **already mounted** (e.g. your system disk on sda) This is considered
> too dangerous, so the developers stopped this in **RHEL6** . You can work around this
> (although not advisable) by using `partx` to forcibly add new partitions to `/proc/partitions`

上面帖子说 RHEL6 中 `partprobe` 不支持更新 **已挂载** 的磁盘设备，要用 `partx` 命令 **强制** 更新分区表：

    $ man partx

       -a, --add
       Add the specified partitions, or read the disk and add all partitions.

    # partx -av /dev/vda
    device /dev/vda: start 0 size 2097152000
    gpt: 3 slices
    # 1:      2048-   411647 (   409600 sectors,    209 MB)
    # 2:    411648- 83884031 ( 83472384 sectors,  42737 MB)
    # 3:  83884032-251656191 (167772160 sectors,  85899 MB)
    BLKPG: Device or resource busy
    error adding partition 1
    BLKPG: Device or resource busy
    error adding partition 2
    added partition 3

上面输出的 `BLKPG: Device or resource busy，error adding partition 1` 报错没关系。
因为前两个分区已经 **被识别挂载** 了，就不用再 **画蛇舔脚** 的重复添加到分区表了。
执行完后，就可以在 `/proc/partitions` 看到新识别的分区了

    # cat /proc/partitions
    major minor  #blocks  name

     252        0  125829120 vda
     252        1     204800 vda1
     252        2   41736192 vda2
     252        3   83886080 vda3

### 扩容文件系统

修复完分区表、更新完分区表。是该做正经事的时候了：**扩容文件系统**

- 将新分区，添加物理卷
- 扩容逻辑卷组
- 拉伸逻辑卷
- 扩容文件系统

doit

    + pvcreate /dev/vda3
      Writing physical volume data to disk "/dev/vda3"
      Physical volume "/dev/vda3" successfully created

    + vgextend vg /dev/vda3
      Volume group "vg" successfully extended

    + lvextend -l +100%FREE /dev/vg/home
      Extending logical volume home to 87.75 GiB
      Logical volume home successfully resized

    + resize2fs -p -F /dev/vg/home
    Filesystem at /dev/vg/home is mounted on /home; on-line resizing required
    old desc_blocks = 1, new_desc_blocks = 6
    Performing an on-line resize of /dev/vg/home to 23003136 (4k) blocks.
    The filesystem on /dev/vg/home is now 23003136 blocks long.

## OVER

到此就完成了，虚拟机扩容文件系统的任务就完成了。。。



<br/>

本文标题 | [{{ page.title }}]({{ page.url }})
-------- |:--------
原始链接 | <{{ site.url }}{{ page.url }}>
