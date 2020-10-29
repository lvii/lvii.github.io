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

# HOW

搜了一下发现是 `open-vm-tools` 的 bug 问题：

[Help with display size in fedora VMWare guest: it won't maximise to 4k screen 2020-10-23](https://www.reddit.com/r/Fedora/comments/jg9k5g/help_with_display_size_in_fedora_vmware_guest_it/)

    cp -v /etc/vmware-tools/tools.conf.example /etc/vmware-tools/tools.conf
    sed -i '/^\[resolutionKMS/a enable=true' /etc/vmware-tools/tools.conf

修改 `/etc/vmware-tools/tools.conf` 配置文件后，重启 `vmtoolsd` 服务，虚拟机的分辨率立马自动适配窗口了







