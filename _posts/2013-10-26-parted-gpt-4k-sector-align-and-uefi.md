---
layout: post
title: GPT 分区 4k 扇区对齐和 UEFI 引导
category: system
tags: [disk,gpt,uefi]
---

换了 Thinkpad x230 本子，硬盘已 **高级格式化** 过，是 4k 扇区的

小黑的 **mini PCI-E** 接口支持 **mSATA** 就再加了一条 64G 的东芝 SSD

磁盘使用 GPT 分区表，系统引导也换用 UEFI 模式

- HDD 上面安装 arch 然后装了个 win8
- SSD 上面装的 gentoo

换了新玩法，先上张整理的思维导图，下面是折腾手记：

<http://mind42.com/mindmap/1f733b68-19ac-4431-9ea4-c819328122c8>

![4k align and GPT parted UEFI][img]

## UEFI bootable usb

所有系统的安装需要在 UEFI mode 下面进行，要先搞一个能从 UEFI 启动的 LiveUSB

参考 archlinux wiki : [「Create_UEFI_bootable_USB_from_ISO」][0] 可以简单制作一个启动盘

## 设置 BIOS 引导选项

将 BIOS boot 引导 `UEFI/Legacy BOOT` 修改为 `UEFI ONLY`

插上 U 盘

开机按 **F12** 就可以引导 archlinux LiveUSB 了

## 4k 扇区对齐

对 4k sector 硬盘分区，要实现扇区对齐，可以`parted` 和 `gdisk` 两个分区工具：

命令    | 描述
    --- | ---
`parted`  | 支持 MBR / GPT 分区之间没有间隙
`gdisk`   | 只支持 GPT 分区之间有间隙

### 工具测试

下面分别使用 gdisk 和 parted 测试以下分区大小：

挂载点      | `/boot` | `/`         | `/home`
        --- | ---   | ---       | ---
分区大小    | 200M  | 200M~30G  | 30G~60G

使用 `parted /dev/sda unit s print free` 命令查看两个工具，扇区对齐的结果不同

gdisk 每个分区起始扇区都为 8 的倍数，但在分区之间会留一段 `2047s` 大小的空隙:

    Number  Start       End          Size         File system  Name  Flags
            34s         2047s        2014s        Free Space
     1      2048s       409600s      407553s                   boot
            409601s     411647s      2047s        Free Space
     2      411648s     62914560s    62502913s                 root
            62914561s   62916607s    2047s        Free Space
     3      62916608s   125829120s   62912513s                 home

parted 实现的扇区对齐，分区之间是连续的，分区的起始也为 8 的整数倍:

    Number  Start       End          Size         File system  Name  Flags
            34s         2047s        2014s        Free Space
     1      2048s       391167s      389120s                   boot
     2      391168s     58593279s    58202112s                 root
     3      58593280s   117186559s   58593280s                 home

### parted 分区

**注意：** 使用 parted 进行分区时 **第一个分区** 的起始参数要用 `0%` 而不是 `0` ：

    # parted /dev/sda mklabel gpt mkpart primary 0% 200M

如果使用数字 `0` 作为分区开始，则无法实现分区对齐 parted 会提示警告：

    Warning: The resulting partition is not properly aligned for best performance.

    Number  Start  End      Size     File system  Name  Flags
     1      34s    390625s  390592s               boot
            ^^^
            强行确认，起始扇区是 34s

## ESP (EFI System Partition) 分区

ESP 分区的 3 个配置项：

选项 | 配置
---- | ----
分区类型 ( partition type code ) | `EF00`
分区大小 | `200M` ~ `500M`
文件系统 | FAT32

### 分区类型

ESP 的分区类型是 `EF00` 使用 parted 设置 GPT 分区 `set <ID> boot on`

即可将分区类型设置为 **EF00** 参考自 wiki : [「UEFI Gentoo Quick Install Guide」][1]

> Setting the `boot` flag by `parted` in a **MBR** partition marks that partition as **bootable**,
> while in a **GPT** partition it is marked as **EFI System Partition**

parted 无法看到分区 code：

    # parted /dev/sda print
    ...  ...
    Number  Start   End    Size   File system  Name  Flags
     1      1049kB  200MB  199MB               ESP   boot

gdisk 可以列出分区 code 用来判断是否为正确的分区类型： **EF00**

    # gdisk -l /dev/sda
    ...  ...
    Number  Start (sector)    End (sector)  Size       Code  Name
       1            2048          391167   190.0 MiB   EF00  ESP

### 分区大小

磁盘扇区大小不同 ESP 大小也是不一样的。翻了好多文档，不同的文档有不同的解释。

[「Configure UEFI/GPT-Based Hard Drive Partitions」][2]

文档的 DiskPartitionRules 章节 Note 部分定义的 ESP 分区大小：

> For **Advanced Format 4K Native drives** (4-KB-per-sector) drives, the minimum size is `260 MB`,
> due to a limitation of the **FAT32** file format. The minimum partition size of FAT32 drives is calculated
> as sector size **(4KB) x 65527 = 256 MB**.
> Advanced Format **512e** drives are not affected by this limitation, because their emulated sector size
> is 512 bytes. **512 bytes x 65527 = 32 MB**, which is less than the `100 MB` minimum size for this partition.

引用中的 `65527` 是指 FAT32 文件系统支持的最少的 **簇**（和 Linux 文件系统的 block 同一个概念）

[「Windows XP 中 FAT32 文件系统的限制」][3] 文档解释：

> 使用 FAT32 文件系统时，请注意下列限制：... FAT32 卷必须至少包含 **65527** 个簇

上面解释的 ESP 的大小，应该是针对 windows 系统。UEFI 支持的文件系统 FAT32 毕竟是微软搞的

没有找到 Linux 下对 ESP 大小定义的具体数据，仅从从 gentoo wiki 找到下面一段对 ESP 大小的描述：

<http://gentoo-en.vfose.ru/wiki/UEFI#File_system_support>

> The optimum size of the ESP varies depending on the number of OSes
> you're installing and the Linux boot loader you choose to use.
> In particular, ELILO and the Linux kernel's built-in EFI stub loader both require that
> the Linux kernel and associated RAM disk be stored on the ESP, so you should size the ESP
> much as you'd size a separate Linux `/boot` partition (or larger) -- **a minimum of 200 MiB**,
> with `500 MiB` being a more desirable size. If you use GRUB as your boot loader,
> the kernel can reside on a Linux partition, and the boot loader's needs are more modest,
> so `100-200 MiB` may be a reasonable size. Making your ESP larger than necessary can
> increase your flexibility for future choices.

那对于 linux ESP 分区大小：

- 高级格式化的 4k 扇区硬盘 ESP 分区大小最小可设置为 `256M`
- 高级格式化的 `512e` 和普通的 512 扇区磁盘，ESP 最小可设置为 `100M`

我安装双系统的 HHD 上指定 ESP 大小为 `500M` ；SSD 指定 `300M` 都可以正常引导

HDD:

    Number  Start   End     Size    File system  Name   Flags
     1      1049kB  500MB   499MB   fat32        ESP    boot

SSD:

    Number  Start   End     Size    File system  Name   Flags
     1      1049kB  300MB   299MB   fat32        ESP    boot

## UEFI 引导

### EFI stub boot 直接引导

**EFI Boot Stub** 可以直接通过 UEFI 加载内核，引导系统 [EFI System partition][4] 的相关描述：

> **EFI Boot Stub** makes it possible to boot a Linux kernel without  
> the use of a conventional UEFI boot loader, such as GRUB2 or elilo.

gentoo wiki : [「EFI_stub_kernel」][5] 提到首先要启用 `CONFIG_EFI_STUB` 等相关 kernel 配置

**注意** ：使用 EFI stub boot 需要将 root 分区路径，以及其他内核参数通过 `CONFIG_CMDLINE` 变量编译进 kernel

> UEFI does not pass **kernel parameters** to the kernel during normal boot, so you need to
> **hardcode** them via `CONFIG_CMDLINE`. Example for the root partition on `/dev/sda2`.

### GRUB 引导

可以使用 UEFI 引导 grub 然后通过 grub 加载内核，引导系统。和之前 BIOS/MBR 模式类似

相比上面的 EFI boot stub 引导方式，使用 grub 引导 kernel 可以随时修改 kernel 的引导参数

比如 **进单用户模式** 、 **修改 SSD 的磁盘调度策略** 等 ……

EFI 模式下 gentoo 安装 grub 已经自动化了，基本不需要手动安装配置了，[「GRUB2 wiki」][6] 中的安装操作步骤，
看完了让人凌乱了，有些操作是之前 **手动安装** 的方式，而且文档中的路径也很乱，明显是不同人编辑 wiki 时，
使用的是自己的安装的路径，大家越改越乱 `-_-#` 下面是我的安装操作：

### 自动安装 GRUB

    # echo 'GRUB_PLATFORMS="efi-64"' >> /etc/portage/make.conf

    # mkdir -pv /boot/efi/EFI/BOOT
    mkdir: created directory '/boot/efi/EFI'
    mkdir: created directory '/boot/efi/EFI/BOOT'

安装 grub2 时，emerge 会自动安装依赖 `sys-boot/efibootmgr`

    # emerge -avj grub:2

`grub2-install` 安装时，会自动调用 `efibootmgr` 添加引导项，都已自动化，无需额外补刀了

    # grub2-install --target=x86_64-efi --efi-directory=/boot/efi /dev/sdb
    BootCurrent: 0018
    Timeout: 0 seconds
    BootOrder: 001A,0000,0001,0002,0003,000D,0018,0019,000A,000C,0007,0008,0009,000B,000E,000F,0010,0011,0012
    Boot0000  Setup
    Boot0001  Boot Menu
    ...
    Boot0018* arch_grub
    Boot0019* Windows Boot Manager
    Boot001A* gentoo
    Installation finished. No error reported.

根据 kernel 生成配置文件：

    # grub2-mkconfig -o /boot/grub/grub.cfg

下面是安装好 grub 后 `/boot` 下相关的文件目录结构：

    $ tree -F /boot/efi
    /boot/efi
    └── EFI/
        ├── BOOT/       ## BOOT 目录为空，没有用到
        └── gentoo/
            └── grubx64.efi*

    $ tree -F /boot/grub/
    /boot/grub/
    ├── grub.cfg
    ├── grubenv
    ├── locale/
    │   ├── ...
    │   └── zh_CN.mo
    └── x86_64-efi/
        ├── core.efi
        ├── grub.efi
        ├── fat.mod
        ├── ext2.mod
        ├── part_gpt.mod
        ...

`/boot/efi/EFI/gentoo/grubx64.efi` 和 `/boot/grub/x86_64-efi/core.efi` 是同一个文件 `grubx64.efi` 有执行权限而已

    # find /boot -name '*.efi'|xargs md5sum
    29075033ec7ac6834560657d8251370a  /boot/efi/EFI/gentoo/grubx64.efi
    29075033ec7ac6834560657d8251370a  /boot/grub/x86_64-efi/core.efi
    d214e057be029b1ac965401c2db2d0b4  /boot/grub/x86_64-efi/grub.efi

如果是 **手动安装** grub 要自己将 `/boot/grub/x86_64-efi/core.efi` 复制到 `/boot/efi/EFI/` 下的

之后重启系统，在 BIOS boot 设置中，可以看到刚安装的 gentoo 引导项

设置为默认后，进入系统，使用 `efibootmgr` 可以查看引导顺序已是 `001A` 对应的是 gentoo

    # efibootmgr
    BootCurrent: 001A
    Timeout: 0 seconds
    ...

`efibootmgr` 命令也可以修改引导顺序，还没测试，后面有机会，试试看。基本折腾完了。

[0]: https://wiki.archlinux.org/index.php/Unified_Extensible_Firmware_Interface#Create_UEFI_bootable_USB_from_ISO
[1]: http://wiki.gentoo.org/wiki/UEFI_Gentoo_Quick_Install_Guide
[2]: http://technet.microsoft.com/en-us/library/hh824839.aspx#DiskPartitionRules
[3]: http://support.microsoft.com/kb/314463
[4]: http://en.wikipedia.org/wiki/EFI_System_partition
[5]: http://wiki.gentoo.org/wiki/EFI_stub_kernel
[6]: https://wiki.gentoo.org/wiki/GRUB2
[img]: http://media-cache-ec0.pinimg.com/originals/f1/5f/d8/f15fd8ffb572582a886859666f10e172.jpg

<br/>

本文标题 | [{{ page.title }}]({{ page.url }})
-------- |:--------
原始链接 | <{{ site.url }}{{ page.url }}>
