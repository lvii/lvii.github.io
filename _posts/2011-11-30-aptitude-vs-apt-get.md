---
layout: post
title: aptitude 和 apt-get 的一些不同
category: system
tags: [apt, aptitude, debian]
---

发行版 | 软件包管理工具
debian | `aptitude`
ubuntu | `apt-get`、`apt-cache` ...

一直纠结于两者的区别 google 很多，各种说法，看来要求甚解，要研读 man + 官方文档

[debian 官方手册](http://www.debian.org/doc/manuals/debian-reference/ch02.en.html#_literal_apt_get_literal_literal_apt_cache_literal_vs_literal_aptitude_literal) 对比，
包含 `apt / aptitude` 功能对比，还有两者的常用命令对应表格

下面列一下，找出来的一些不同:

## 速度

`apt` 比较轻巧，速度比较快，资源占用较少：

    apt-cache search python
    aptitude search python

对比速度马上见分晓，快的代价：

- `apt-cache` 搜索出来的格式 **可读性** 较差
- `aptitude` 查询结果 **文本对齐**，而且还有安装标记

`aptitude` 虽然较 `apt` 有些大，有些稍慢，但是它的命令统一用 `aptitude` 管理

不像 `apt` 那样分散成多个工具实现不同功能 ( `apt-get` / `apt-cache` / `apt-config` ... )

还有个喜欢 `aptitude` 原因：虽然字符上面 `aptitude` 比 `apt-get` 多一个，
但是，输入 `apti TAB` 比 `apt- TAB` 懒些，虽然都用 `alias` 把常用操作设置了别名

## 依赖

说到软件管理，不得不提 "依赖"

`apt` 依赖查询比较简洁明了：

    # apt-cache depends aptitude
    aptitude
      Depends: libapt-pkg4.11
      Depends: libboost-iostreams1.46.1
      Depends: libc6
      Depends: libcwidget3
      Depends: libept1
      Depends: libgcc1
      Depends: libncursesw5
      Depends: libsigc++-2.0-0c2a
      Depends: libsqlite3-0
      Depends: libstdc++6
      Depends: libxapian22
     |Suggests: aptitude-doc-en
      Suggests: <aptitude-doc>
        aptitude-doc-cs
        aptitude-doc-en
        aptitude-doc-es
        aptitude-doc-fi
        aptitude-doc-fr
        aptitude-doc-ja
      Suggests: tasksel
      Suggests: debtags
      Recommends: sensible-utils
      Recommends: apt-xapian-index
      Recommends: libparse-debianchangelog-perl
      Conflicts: <ia32-apt-get>
      Conflicts: <ia32-apt-get:i386>
      Conflicts: aptitude:i386

还有一个 **反查** 依赖的命令：`rdepends`

    # apt-cache rdepends aptitude
    aptitude
    Reverse Depends:
      aptitude:i386
      dpkg:i386
      wajig
      ppa-purge
      pkgsync
      ibid
      febootstrap
      fai-server
      daptup
      aptitude-gtk
      aptitude-doc-ja
      aptitude-doc-fr
      aptitude-doc-fi
      aptitude-doc-es
      aptitude-doc-cs
      apt-watch-gnome
      apt-dater-host
      tasksel
      dpkg
      aptitude-doc-en
     |aptitude-dbg
     |apt

`aptitude` 有个类似的 `aptitude why | why-not` 但是没有 `apt` 的这个直观简洁

    # aptitude why apt
    i   unattended-upgrades Depends apt

    # aptitude why aptitude
    i   apt Suggests aptitude | synaptic | wajig

    # aptitude why-not apt
    p   dpkg Provides  dpkg
    p   dpkg Suggests  apt
    p   apt  Conflicts apt

    # aptitude why-not aptitude
    p   dpkg     Provides  dpkg
    p   dpkg     Suggests  apt
    p   apt      Suggests  aptitude | synaptic | wajig
    p   aptitude Conflicts aptitude

关于 `install` / `purge` / `clean` 对依赖的处理 `man` 里面没有细说到依赖解决到什么地步

上面的 `apt-cache depends aptitude` 依赖查询，差不多涵盖了所有的依赖情形：

    Depends: libapt-pkg4.11
    ...
    Suggests: <aptitude-doc>
    ...
    Recommends: sensible-utils
    ...
    Conflicts: aptitude:i386
    ...

`apt` 和 `aptitude` 只要是 `depends` 的依赖，都会安装！

建议 / 推荐 什么的 `aptitude` 和 `apt` 没有自动安装过

## 卸载

这位兄台的 blog 对卸载，依赖处理做了 **通俗易懂图文并茂** 的解释：+1024

[`apt-get remove` 与 `apt-get autoremove`、`aptitude remove` 的不同](http://www.igigo.net/post/archives/88)

- `apt autoremove` 卸载不再需要的依赖，不知道 `autoremove` 怎么处理配置文件 [?]
- `aptitude purge` 会清除不需要的依赖，没用过 `apt purge` 测试过

再就是两个 **版本升级** 有些不同：

    aptitude safe-upgrade    apt-get upgrade
    aptitude full-upgrade    apt-get dist-upgrade

## 清理

最后就是 `clean` 和 `autoclean`

`clean` 比 `autoclean` 打扫的干净些，只要是缓存 `clean` 通通删除

`autoclean` 会留最近一次升级的缓存，像内核，驱动什么的，如果新版不稳定，可以回退到上个版本














<br/>

本文标题 | [{{ page.title }}]({{ page.url }})
-------- |:--------
原始链接 | <{{ site.url }}{{ page.url }}>
