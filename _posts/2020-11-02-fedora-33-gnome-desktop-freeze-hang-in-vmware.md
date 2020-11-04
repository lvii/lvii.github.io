---
layout: post
title: "解决 VMware Fedora 33 虚拟机 Gnome 桌面 hang 住问题"
category: desktop
tags: [fedora,vmware,gnome]
---

# WHAT

在 VMware 使用 gnome 桌面会出现卡住问题，开始以为是死机了，但当执行关机操作，发现虚拟机能正常响应关机信号。看来不是死机，后面再出现类似 hang 住问题，通过 ssh 登录排查了一下原因。

# WHY

通过 `top` 发现 `Xwanland` CPU 占用 100% 原来是 gnome 桌面挂了，内核和底层服务都正常的

    # top -bc -n 1|head
    top - 22:27:07 up 2 min,  2 users,  load average: 1.03, 0.50, 0.20
    Tasks: 260 total,   2 running, 258 sleeping,   0 stopped,   0 zombie
    %Cpu(s): 26.1 us,  1.4 sy,  0.0 ni, 71.0 id,  0.0 wa,  1.4 hi,  0.0 si,  0.0 st
    MiB Mem :   3900.6 total,   2164.1 free,    853.2 used,    883.3 buff/cache
    MiB Swap:   1950.0 total,   1950.0 free,      0.0 used.   2798.8 avail Mem

     PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND
    1441 bob       20   0 1057200  63344  44388 R 100.0   1.6   1:50.92 /usr/bin/Xwayland :0 -rootless -noreset -accessx -core -auth /run/user/1000/.mutter-Xwaylandauth.TG7US0 -listen 4 -displayfd 5 -listen 6 -extension SECURITY
    2123 root      20   0   19384   4876   4084 R   6.2   0.1   0:00.01 top -bc -n 1
       1 root      20   0  171256  13676  10440 S   0.0   0.3   0:02.58 /usr/lib/systemd/systemd --switched-root --system --deserialize 30

# HOW

通过重启 `gdm` 服务，重启 Gnome 桌面环境

    # service gdm restart

网上查了一下发现这个问题还很普遍呢

[GNOME desktop freezing on VMware Workstation Pro for Windows host and Fedora guest (likely due to 3D acceleration) #176](https://github.com/vmware/open-vm-tools/issues/176)

解决方法也比较简单，在 VMware 虚拟机【隔离】设置 **禁用** 鼠标 **拖放功能**

![img](https://i.imgur.com/tui3ocy.png)
