---
layout: post
title: "VMware 设置 Fedora 33 Gnome 桌面 1080p 分辨率"
category: desktop
tags: [fedora,vmware,gnome]
---

# WHAT

Fedora 33 最近两天刚发布，在 VMware 下安装想体验一下 Workstation 版本的 Gnome 桌面

结果发现 Gnome 桌面设置里的分辨率没有 `1920x1080`

这有点不科学。。。

# WHY

搜了一下发现是 `open-vm-tools` 的 bug 问题：

    $ rpm -qf /etc/vmware-tools/tools.conf.example
    open-vm-tools-11.1.5-1.fc33.x86_64

[Bug 1890815 - Wayland session as vmware 16 guest does not resize or maximise screen](https://bugzilla.redhat.com/show_bug.cgi?id=1890815)

> The issue is that `libresolutionKMS.so` is **NOT** loaded into the Wayland session on **Fedora 33** (tested w/ **VMWare Workstation 16**). It works as expected in Ubuntu 20.10 with either Wayland or Xorg.
>
>     cp /etc/vmware-tools/tools.conf.example /etc/vmware-tools/tools.conf
>     nano /etc/vmware-tools/tools.conf
>
> Remove the `#` from this block:
>
>     [resolutionKMS]
>
>     # Default is true if tools finds an xf86-video-vmware driver with
>     # version >= 13.2.0. If you don't have X installed, set this to true manually.
>     # This only affects tools for Linux.
>     enable=true
>
>     systemctl restart vmtoolsd.service

# HOW

修改 `/etc/vmware-tools/tools.conf` 配置文件后

    cp -v /etc/vmware-tools/tools.conf.example /etc/vmware-tools/tools.conf
    sed -i '/^\[resolutionKMS/a enable=true' /etc/vmware-tools/tools.conf
    systemctl restart vmtoolsd

重启 `vmtoolsd` 服务，虚拟机的分辨率立马自动适配窗口了







