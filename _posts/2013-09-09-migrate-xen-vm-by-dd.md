---
layout: post
title: 使用 dd 在线迁移 xen 虚拟机
category: system
tags: [xen, virt]
---

## 经过：

厂里有两台在旧的 xen 宿主机上面的虚拟机要迁移到新的 xen 宿主机

旧的 xen 虚拟机的存储使用的是 **物理分区**，新的 xen 虚拟机使用的是 **LVM 逻辑卷**

google 了一圈，关于 xen 的迁移大部分都是基于 img 文件格式的，场景不一样，差评啊！

[「xen 虚拟机的迁移类型」](http://www.mysqlops.com/2012/05/24/xenmigratetype.html)

上面的文章使用 `xm migrate` 命令进行迁移的两种方式：

- 静态迁移，虚拟机休需要眠掉，会影响在线业务
- 动态迁移，进行迁移的宿主机的环境要做一些配置，还要用到共享存储

有些高端，又搜了一圈，在 xen 的邮件列表中，有位兄台提到用 dd 醍醐灌顶。。。

<http://lists.xen.org/archives/html/xen-users/2013-01/msg00264.html>

那就把原来的物理分区 dd 打包一份，然后再 dd 到要迁移的宿主机上面。

当前虚拟机的磁盘是在 **物理分区** : `/dev/cciss/c0d0p8`

    # parted /dev/cciss/c0d0p8 print

    Model: Unknown (unknown)
    Disk /dev/cciss/c0d0p8: 197GB
    Sector size (logical/physical): 512B/512B
    Partition Table: msdos

    Number  Start   End     Size    Type      File system  Flags
     1      32.3kB  132MB   132MB   primary   ext3         boot
     2      132MB   4326MB  4195MB  primary   ext3
     3      4326MB  6473MB  2147MB  primary   linux-swap
     4      6473MB  197GB   191GB   extended
     5      6473MB  17.0GB  10.5GB  logical   ext3
     6      17.0GB  21.2GB  4195MB  logical   ext3
     7      21.2GB  25.4GB  4195MB  logical   ext3
     8      25.4GB  29.5GB  4195MB  logical   ext3
     9      29.5GB  197GB   167GB   logical   ext3

`dd` 的时候用管道给 `gzip` 压缩了一下，有些耗时

    # time dd if=/dev/cciss/c0d0p8 |gzip > /media/vm6.img
    384772752+0 records in
    384772752+0 records out
    197003649024 bytes (197 GB) copied, 6417.23 seconds, 30.7 MB/s

    real    106m57.278s
    user    103m13.547s
    sys     6m22.712s

`dd` 完后，再把当前 vm 的配置信息导出一下：`virsh dumpxml vm6 > vm6.xml`

把 `dd` 好的文件和 vm 的配置文件 `scp` 到目的宿主机上，由于新的宿主机使用的是 **逻辑卷**

创建的逻辑卷大小 (197GiB) 比原来的磁盘分区 (197GB) 大了点，没有小

    # lvcreate -L 197GB -n vm_disk3 vg
      Logical volume "vm_disk3" created

然后把镜像文件解压 dd 灌进新创建的 LV 貌似也比较耗时：

    # time gzip -dc vm6.img | dd of=/dev/vg/vm_disk3
    384772752+0 records in
    384772752+0 records out
    197003649024 bytes (197 GB) copied, 6170.47 seconds, 31.9 MB/s

    real    102m50.593s
    user    12m42.816s
    sys     13m36.711s

[使用 dd 和 gzip 代替 ghost 做磁盘镜像](http://linuxtoy.org/archives/make_disk_image.html)

这篇文章的评论中，有位兄台提到 dd 时指定 bs 写入速度会快些，下次有机会测试一下

修改 `vm6.xml` 中定义的虚拟机的磁盘路径：

    <disk type='block' device='disk'>
      <driver name='phy'/>
      <source dev='/dev/vg/vm_disk3'/>
      <target dev='xvda' bus='xen'/>
    </disk>

然后 `virsh define vm6.xml` 定义新的 VM

虚拟机中的一些配置文件还需要修改一下：

    /boot/grub/menu.lst
    /etc/fstab

这 2 个文件中可能会用磁盘 `UUID` 来 mount 文件系统，迁移之后，虚拟机分区的 `UUID` 变了：

    原系统
    # dumpe2fs -h /dev/xvda1|grep -i uuid
    Filesystem UUID:          f9c79e8b-2945-4f37-b8b7-ef14538a9397

    迁移后系统
    # dumpe2fs -h /dev/xvda1|grep -i uuid
    Filesystem UUID:          9a906e6e-f13e-48ad-b2be-9b5e91364eac

还有就是 IP 地址和 mac 地址，mac 地址在刚才 define 之前没有改，复用原来的
IP 地址的配置文件在虚拟机内部，需要挂载虚拟机的文件系统，使用 `kpartx` 来搞：

    # parted /dev/vg/vm_disk1 print

    Model: Linux device-mapper (dm)
    Disk /dev/mapper/vg-vm_disk1: 212GB
    Sector size (logical/physical): 512B/512B
    Partition Table: msdos

    Number  Start   End     Size    Type      File system  Flags
     1      32.3kB  132MB   132MB   primary   ext3         boot
     2      132MB   4326MB  4195MB  primary   ext3
     3      4326MB  6473MB  2147MB  primary   linux-swap
     4      6473MB  197GB   191GB   extended
     5      6473MB  17.0GB  10.5GB  logical   ext3
     6      17.0GB  21.2GB  4195MB  logical   ext3
     7      21.2GB  25.4GB  4195MB  logical   ext3
     8      25.4GB  29.5GB  4195MB  logical   ext3
     9      29.5GB  197GB   167GB   logical   ext3

使用 `kpartx` 来创建虚拟机的磁盘映射关系

    # kpartx -av /dev/vg/vm_disk1
    add map vm_disk1p1 : 0 256977 linear /dev/vg/vm_disk1 63
    add map vm_disk1p2 : 0 8193150 linear /dev/vg/vm_disk1 257040
    add map vm_disk1p3 : 0 4192965 linear /dev/vg/vm_disk1 8450190
    add map vm_disk1p5 : 0 20482812 linear /dev/vg/vm_disk1 12643218
    add map vm_disk1p6 : 0 8193087 linear /dev/vg/vm_disk1 33126093
    add map vm_disk1p7 : 0 8193087 linear /dev/vg/vm_disk1 41319243
    add map vm_disk1p8 : 0 8193087 linear /dev/vg/vm_disk1 49512393
    add map vm_disk1p9 : 0 327051207 linear /dev/vg/vm_disk1 57705543

挂载修改配置：

    # mount /dev/mapper/vm_disk1p2 /media   ## 根分区
    # vim /media/etc/sysconfig/network-scripts/ifcfg-eth0

卸载文件系统，移除映射关系：

    # umount /media/
    # kpartx -dv /dev/vg/vm_disk1
    del devmap : vm_disk1p1
    del devmap : vm_disk1p2
    del devmap : vm_disk1p3
    del devmap : vm_disk1p5
    del devmap : vm_disk1p6
    del devmap : vm_disk1p7
    del devmap : vm_disk1p8
    del devmap : vm_disk1p9

现在就可以启动 vm 了：

    # virsh start vm6
    Domain vm6 started

## 结果：

正常开机，磁盘自检，挂载，网络等配置等都正常，没有比较明显的后遗症 OVER ...

## 起因：

要迁移的 vm 是因为厂里的应用运维不想搞业务迁移，说上面的应用太老，服务配置麻烦，还有很多坑，
所以不想迁移业务，闹着要整体迁移虚拟机，好吧，只好我来填坑喽 。。。

## 感言：

木桶原理啊：那些边缘业务，没人维护，没人关心，一旦发生变化，就成了短板，变成了需要花精力和时间来维护的坑



<br/>

本文标题 | [{{ page.title }}]({{ page.url }})
-------- |:--------
原始链接 | <{{ site.url }}{{ page.url }}>
