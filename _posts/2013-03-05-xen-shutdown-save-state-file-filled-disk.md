---
layout: post
title: "xen 宿主机重启后 xm li 提示 Error: (28, 'No space left on device')"
category: system
tags: [xen, redhat, virt]
comments: true
---

有台 xen 宿主机重启之后 xm 相关的命令都失效了，无法获得 vm 的状态

## 现象

    # xm li
    Error: (28, 'No space left on device')
    Usage: xm list [options] [Domain, ...

    # virsh list --all
    error: Failed to list inactive domains
    error: got unknown HTTP error code 1862947920

## 定位

提示应该和磁盘空间有关系 `df -h` 查看根分区被占满了，需要找出是什么文件占用了空间：

    # df -h
    Filesystem            Size  Used Avail Use% Mounted on
    /dev/sda2              15G   15G     0 100% /
    /dev/sda1             200M   40M  161M  20% /boot

    # du -sh /*
    9.6G    var
    
    # du -sh /var/lib/xen/*
    4.0K    images
    9.5G    save
    16K     xend-db
    
    # ll /var/lib/xen/save
    total 9876528
    -rwxr-xr-x 1 root root     2428928 Feb 25 09:31 vm04
    -rwxr-xr-x 1 root root 10094542848 Feb 25 09:31 vm02
    -rwxr-xr-x 1 root root     4878336 Jan 29 19:04 vm03
    -rwxr-xr-x 1 root root     1818624 Jan 29 19:05 vm05

## 清空

在 `/var/lib/xen/save` 目录下有各个 vm 的状态文件占满了文件夹，需要清空：

    # cd /var/lib/xen/save
    # rm -f *

    # df -h
    Filesystem            Size  Used Avail Use% Mounted on
    /dev/sda2              15G  4.7G  8.8G  35% /
    
    # xm li
    Name                                        ID   Mem VCPUs      State   Time(s)
    Domain-0                                     0  3000    24     r-----     77.7
    vm04                                         5  4000     4     -b----      5.1
    ...
    
清空 `/var/lib/xen/save` 目录后 xm 相关命令恢复，根分区被占空间也被释放了

## 根除

为了防止以后 vm 关机后，再保存运行状态，需要修改配置文件，取消该状态保存特性  
以后关机就是真正的 shutdown 了，而不是类似休眠保存运行状态这种模式

    ## Type: string
    ## Default: /var/lib/xen/save
    #
    # Directory to save running domains to when the system (dom0) is
    # shut down. Will also be used to restore domains from if # XENDOMAINS_RESTORE
    # is set (see below). Leave empty to disable domain saving on shutdown
    # (e.g. because you rather shut domains down).
    # If domain saving does succeed, SHUTDOWN will not be executed.
    #
    #XENDOMAINS_SAVE=/var/lib/xen/save
    XENDOMAINS_SAVE=

修改完成后，重启服务：

    # /etc/init.d/xendomains restart

## 参考

[Remove of SAVE state on XEN virtual machine][1]  
[Shutting down xen domUs without saving them][2]

[1]: http://scottcoats.blogspot.com/2008/04/remove-of-save-state-on-xen-virtual.html
[2]: http://raftaman.net/?p=237

<br/>

本文标题 | [{{ page.title }}]({{ page.url }})
-------- |:--------
原始链接 | <{{ site.url }}{{ page.url }}>
