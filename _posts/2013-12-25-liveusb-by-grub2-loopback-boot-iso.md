---
layout: post
title: grub2 引导 iso 制作 liveusb 启动盘
category: system
tags: [grub, liveusb]
---

很久之前装 Linux 时，做 USB 启动盘，比较麻烦。要用 unetbootin，usb creater 之类的工具。
arch 虽然简单些，可以直接 `dd` 。但是这些方法都有个痛点，就是要把 u 盘给格式化，或是覆盖擦写。
GRUB2 的 **loopback** 特性，可以直接引导 ISO 文件，只要给 U 盘安装个 grub 配置一下就 OK 了。

相比传统制作启动盘的方式：

- 不用格式化 U 盘
- grub 可以引导 U 盘中，不同的发行版的 ISO
- ISO 更新，只要替换 ISO 文件就可以，依旧不用再折腾
- 不影响 U 盘的正常使用

废话不多说，开搞：

## 分区

### 分区起始位置

如果 U 盘之前的 **分区起始位置** 太小，最好删除分区，重新调整一下分区的起始扇区。

因为 grub 要在 MBR 和第一个分区之间，安装 `stage1.5` 即 `core.img` 所以分区起始前的空白区域不宜太小。

下面是引用 **维基百科** ( 原文图文并茂 )：<http://en.wikipedia.org/wiki/GNU_GRUB>

> **Stage 1.5**:
> `core.img` is by default written to the sectors between the **MBR** and the **first partition**, when these
> sectors are free and available. For legacy reasons, the first partition of a hard drive does not begin at
> sector `1` (counting begins with `0`) but at sector `63`, leaving a gap of `63` sectors of empty space.

用 parted 分区时，起始位置指定 `0%` ，即第 `2048s` 作为分区的起始扇区：

    · sudo parted /dev/sdb mkpart primary 0% 100%

### 分区 bootable

**不要忘了** 设置分区的 `boot` 属性：

    · sudo parted /dev/sdb set 1 boot on             ## <-- bootable

    · sudo parted /dev/sdb print
    Model: SanDisk Cruzer Switch (scsi)
    Disk /dev/sdc: 8004MB
    Sector size (logical/physical): 512B/512B
    Partition Table: msdos
    Disk Flags:

    Number  Start   End     Size    Type     File system  Flags
     1      1049kB  8004MB  8003MB  primary  fat32        boot

### 格式化分区

     mkfs.vfat /dev/sdb1

## 安装 grub

**挂载** u 盘后，安装 grub

    # mount /dev/sdb/ /mnt

    # grub-install --root-directory=/mnt/ --no-floppy /dev/sdb

成功安装好 grub 后，在 u 的根目录出现 `boot` 目录，目录结构如下：

    · ls boot/grub
    fonts/  grubenv  i386-pc/  locale/  themes/

    · tree -F boot
    boot
    └── grub/
       ├── fonts/
       │   └── unicode.pf2
       ├── grubenv
       ├── i386-pc/
       │   ├── ...
       │   ├── loopback.mod
       ... ...
       ├── locale/
       │   ├── ast.mo
       ... ...
       │   └── zh_TW.mo
       └── themes/
            ...

## 配置 grub

安装好 grub 但还没有 `grub.cfg` 此时 U 盘是可以启动的，但只是会进入 grub shell 命令行。

在 u 盘根目录创建 `iso` 文件夹，将下载的 SysrescueCD，Archlinux，Ubuntu ISO 文件复制进去

然后配置 grub.cfg **注意** 文件路径是：`/boot/grub/grub.cfg` **不是** `/boot/grub.conf`

下面是我的 `grub.cfg` 配置，不同的发行版最好是参考 **官方 wiki** 或 帮助文档来配置：

    set timeout=5
    ## 默认启动 sysrescueCD
    set default=1

    set root=(hd0,1)
    set sysrescuecd_iso="/iso/systemrescuecd-x86-3.8.1.iso"
    set archlinux_iso="/iso/archlinux-2013.12.01-dual.iso"
    set ubuntu_server_iso="/iso/ubuntu-12.04.3-server-amd64.iso"

    ## 加载 loopback 模块
    insmod loopback

    menuentry "SystemRescueCd 64bit memory cached" {
        load_video
        set gfxpayload=keep
        echo "Loopback iso file: $sysrescuecd_iso"
        loopback loop $sysrescuecd_iso
        linux (loop)/isolinux/rescue64 isoloop=$sysrescuecd_iso setkmap=us docache
        echo 'Loading Linux core repo kernel ...'
        initrd (loop)/isolinux/initram.igz
        echo 'Loading initial ramdisk ...'
    }

    menuentry "Arch Linux 2013.12.01 x86_64" {
        load_video
        set gfxpayload=keep
        loopback loop $archlinux_iso
        echo "Loopback iso file: $archlinux_iso"
        ## NOTE 'img_dev=' parameter according to your volume label
        linux (loop)/arch/boot/x86_64/vmlinuz archisolabel=ARCH_201312 img_dev=/dev/disk/by-uuid/0303-0336 img_loop=$archlinux_iso
        echo 'Loading Linux core repo kernel ...'
        initrd (loop)/arch/boot/x86_64/archiso.img
        echo 'Loading initial ramdisk ...'
    }

    menuentry "Install Ubuntu Server 12.04 server in expert mode" {
        load_video
        set gfxpayload=keep
        loopback loop (hd0,1)$ubuntu_server_iso
        echo "Loopback iso file: $ubuntu_server_iso"
        linux   (loop)/install/vmlinuz  file=/cdrom/preseed/ubuntu-server.seed iso-scan/filename=$isofile priority=low --
        echo 'Loading Linux core repo kernel ...'
        initrd  (loop)/install/initrd.gz
        echo 'Loading initial ramdisk ...'
    }

到此，多合一 LiveUSB 启动盘就做好了。

- 重启机器
- 选择从 U 盘启动
- 引导进入 grub

选择不同菜单，来加载不同 ISO 镜像

## 参考

[利用 grub2 直读 iso 镜像 制作多启动 u 盘][0]

[0]: http://blog.zforzelda.com/2012/09/grub2-multi-iso-usb-boot-stick.html

[製作直讀光碟 ISO 映像檔多重開機 USB 救急工具碟][1]

[1]: http://ghostsinthelab.org/2696/製作直讀光碟-iso-映像檔多重開機-usb-救急工具碟/

<http://jarodlau.blogspot.com/2011/10/grub2u.html>

<http://www.gnu.org/software/grub/manual/html_node/loopback.html#loopback>

<http://www.gnu.org/software/grub/manual/grub.html#Configuration>

<http://wiki.gentoo.org/wiki/GRUB2#ISO_images>

<https://wiki.archlinux.org/index.php/USB_Installation_Media>

<https://wiki.archlinux.org/index.php/GRUB#Booting_an_ISO_directly_from_GRUB>

<https://help.ubuntu.com/community/Grub2/ISOBoot>

<br/>

本文标题 | [{{ page.title }}]({{ page.url }})
-------- |:--------
原始链接 | <{{ site.url }}{{ page.url }}>
