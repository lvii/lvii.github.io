---
layout: post
title: KVM 下 Windows XP virtio 网卡驱动自动安装失败问题
category: system
tags: [kvm, windows]
---

在 KVM 宿主机上面装了个 Windows XP SP3 测试虚拟机，虚拟机使用 `virtio` 网络驱动

需要在虚拟机上额外安装网卡驱动，使用 **自动安装** 方式，总安装是失败，提示下面错误：

> Windows 无法加载这个硬件的设备驱动程序。驱动程序可能已损坏或不见了。 (代码39)  
> Windows cannot load the device driver for this hardware.  
> The driver may be corrupted or missing. (Code 39).

![xp sp3 kvm virtio network driver auto install failed][a]

[a]: http://media-cache-ec0.pinimg.com/originals/a3/66/c5/a366c578d26a9edf9b8e13aaf2cef930.jpg

网卡无法法正常驱动，识别不到 **本地连接**，网络都没有，好蛋疼，，，啊。

折腾了一圈，发现是安装方法不对。和驱动版本无关，下面是 **淋漓尽致** 的图文手记：

## 镜像挂载

根据 kvm 官网 [「Windows VirtIO Drivers 」][0] 提示，从 fedoraproject 下载最新的 iso 镜像：

[http://alt.fedoraproject.org/pub/alt/virtio-win/latest/images/][url]

[url]: http://alt.fedoraproject.org/pub/alt/virtio-win/latest/images/

在 kvm 宿主机上，将 iso 挂载到虚拟机的 cdrom 光驱上：

    # virsh attach-disk xp ~/virtio-win-0.1-65.iso hdc --type cdrom --mode readonly
    Disk attached successfully

然后，就可以在 xp **我的电脑** 里，通过加载的 iso 光盘安装驱动了。

## 错误安装

使用 xp 默认的 **自动安装** 方式安装 virtio 驱动，就会出现 `(code 39)` 错误。测试 google 到几篇文章，都不行：

- [KVM 下 windows 使用 virtio 驱动][1]  
这位兄台，使用的是 **2009 年** 分离出来的 `virtio-net` 驱动。测试安装了下，还是失败

- [XP SP3 latest VirtIO network & scsi drivers fail to install][2]  
这篇帖子，纠结在驱动的版本上面，换用旧的 `virtio-win-0.1-49.iso` 驱动版本测试了下，还是杯具

- [Windows XP install guide for KVM and OpenStack][3]  
最后，这位大哥，提到了是 **自动安装** 导致的问题：

> Windows will attempt to load the drivers and prompt a warning. Click Continue Anyway  
> - If you just do the automatic method it will install the wrong driver which won't initialize  
> and give you an code: **39 error**

除了 **自动安装** 还有一个 **坑**，驱动目录。ISO 镜像中有两个 `WXP` 和 `XP` 目录

virtio-net 驱动在 **XP 目录** 下，选用 WXP 会提示找不到驱动：

![manual install by hand 5][f]

## 手动安装图解

- 不使用 **自动** 搜索安装，而采用 **手动** 选择驱动路径安装
- virtio-net 驱动路径是 `D:\XP\X86` 而不是 `D:\WXP\X86`

下面是尝试了几次之后，成功安装驱动图解： **从列表或指定位置安装(高级)(S)**

![manual install by hand 1][b]

选择： **不要搜索。我要自己选择要安装的驱动程序(D)**

![manual install by hand 2][c]

通过 `从磁盘安装(H)...` 按钮，选择驱动文件路径：

![manual install by hand 3][d]

选择驱动路径为：`D:\XP\X86` 确定之后，进行安装。忽略提示的驱动 **没有** 数字签名提示：

![manual install by hand 4][e]

安装成功后，查看驱动信息：

![manual install by hand 5][g]

[0]: http://www.linux-kvm.org/page/WindowsGuestDrivers/Download_Drivers
[1]: http://www.361way.com/kvm-windows-virtio/2816.html
[2]: http://forum.proxmox.com/threads/14765-XP-SP3-latest-VirtIO-network-amp-scsi-drivers-fail-to-install
[3]: http://www.unicornclouds.com/blog_posts/kvm_windows_xp_install_openstack

[b]: http://media-cache-ec0.pinimg.com/originals/d6/6a/20/d66a20bcc861846ada8980edf9062f08.jpg
[c]: http://media-cache-ak0.pinimg.com/originals/52/32/10/5232108b8e76311103d3f9caf98b4dcc.jpg
[d]: http://media-cache-ec0.pinimg.com/originals/91/5d/cc/915dccb1c59e6931fcb29ced0896d573.jpg
[e]: http://media-cache-ak0.pinimg.com/originals/0c/07/cc/0c07cc625ee8ee3be792f2c17b633122.jpg
[f]: http://media-cache-ak0.pinimg.com/originals/2d/83/4e/2d834e41fb542fc01c1b21286ca814f5.jpg
[g]: http://media-cache-ec0.pinimg.com/originals/7f/6f/8e/7f6f8e48fd770e02d4e8ec728dcd8ac0.jpg

<br/>

本文标题 | [{{ page.title }}]({{ page.url }})
-------- |:--------
原始链接 | <{{ site.url }}{{ page.url }}>
