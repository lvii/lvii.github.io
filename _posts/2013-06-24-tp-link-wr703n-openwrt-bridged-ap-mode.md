---
layout: post
title: openwrt bridge AP 网络模式
category: network
tags: [openwrt, route]
---

## 准备

- 牙签一根
- [如何进入安全模式 (failsafe mode)](http://wiki.openwrt.org/doc/howto/generic.failsafe)

[「整理的关于 routeos, openwrt, wr703n 的 **思维导图**」][1] openwrt 支持的网络模式：

[1]: http://mind42.com/mindmap/fd387950-16d0-4e0d-b140-7ea1dca8afcf

- [bridged AP](http://wiki.openwrt.org/doc/recipes/bridgedap)
- [route AP](http://wiki.openwrt.org/doc/recipes/routedap)
- [dump AP](http://wiki.openwrt.org/doc/recipes/dumbap)
- client mode (又有好多种)

## bridged AP vs route 模式

### bridged AP 模式

- 相对于上一级路由器的 DHCP server 和 openwrt 无线路由器对下面的接入点是透明的
- 各个接入点通过无线路由器直接向上层的 DHCP 服务器请求地址
- openwrt 上面无需启动多余的 `dnsmasq` 多占一份资源
- (缺点) 没有路由功能，基于上层服务的 VPN， socks 等翻墙代理可能没法用了

### 路由模式

- 多了一层 NAT 环境，分配的地址是 NAT 的 `192.168.x.x` 的地址
- 访问多了一跳路由，效率明显没有二层的 bridge 高

## 访问模式

bridged AP 模式对应 TP Link wr703n **无线 AP 模式** 中的 **接入点(Access point)** 模式

[TL-WR703N 设置指南（三）无线 AP 模式](http://service.tp-link.com.cn/detail_article_201.html)

![tp link wr703 access point bridged AP](http://media-cache-ec3.pinimg.com/originals/e4/94/fb/e494fb74f37af391f7df2d2479841796.jpg)

上图的访问模式：

    (上层 DHCP 服务) --有线--> (openwrt) --无线--> (笔记本，手机)

**bridged AP 官方配置 wiki** ：[http://wiki.openwrt.org/doc/recipes/bridgedap](http://wiki.openwrt.org/doc/recipes/bridgedap)

- 配置网络
- 配置无线
- 关闭 `dnsmasq`

### home

针对家里的上层拨号路由器 `192.168.1.1` 配置：

    config interface 'loopback'
            option ifname 'lo'
            option proto 'static'
            option ipaddr '127.0.0.1'
            option netmask '255.0.0.0'

    config interface 'lan'
            option ifname 'eth0'
            option type 'bridge'
            option proto 'static'
            option ipaddr '192.168.1.11'
            option netmask '255.255.255.0'
            ## 为了 openwrt 可以连接外网，需要配置以下网关和 DNS
            option gateway '192.168.1'
            option dns '192.168.1.1'

奇葩的是 DNS 要由上层路由器 **代理解析**，测试用 google 的 DNS `8.8.8.8` 不行

在家里配置比较简单，注意 **管理地址** 不要冲突就好

### office

办公室的环境和家里的有些不同，还做了限制：

- 办公室的网络是 `10.x.x.x` 网段
- 禁用了 **手动** 设置 IP 访问公网，必须要 DHCP 获取地址

因为是两个网段，需要在 openwrt 配置 **ip 别名 (ip alias)** 进行管理：

**alias 官方配置 wiki** ：<http://wiki.openwrt.org/doc/uci/network#aliases>

    config interface 'loopback'
            option ifname 'lo'
            option proto 'static'
            option ipaddr '127.0.0.1'
            option netmask '255.0.0.0'

    ## 动态获取办公网地址，不然 openwrt 没法上网
    config interface 'lan'
            option ifname 'eth0'
            option type 'bridge'
            option proto 'dhcp'

    ## 手工配置的内网地址，用作管理用
    config 'alias'
            option interface 'lan'
            option proto 'static'
            option ipaddr '10.10.15.187'
            option netmask '255.255.255.0'

    ## 如果上面两个地址分配失败，这个备用地址用来连接管理 openwrt
    ## 需要将笔记本网卡手工设为 192.168.x.x 网段
    config 'alias'
            option interface 'lan'
            option proto 'static'
            option ipaddr '192.168.1.12'
            option netmask '255.255.255.0'

上面的配置，都没有指定 gateway 如果网关 **不是** 真实存且可达的。配置之后，会导致无法登录 openwrt
需要进入安全模式修改配置了。gateway 其实是不需要的，使用网线将路由器和电脑互联，
只要路由器和电脑设为同一网段，掩码一致，就可以直接访问的

    # /etc/init.d/network restart

重启网络后，可以通过手工设置的办公网地址连接到 openwrt 说明配置生效了

登录查看正确的获取到了 DHCP 地址也，但是 `ifconfig` 却无法显示 **ip alias** ：

    # ifconfig
    br-lan    Link encap:Ethernet  HWaddr B0:48:7A:3B:EF:64
              inet addr:10.10.15.38  Bcast:10.10.15.255  Mask:255.255.255.0
              UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
              ... ...

    eth0      Link encap:Ethernet  HWaddr B0:48:7A:3B:EF:64
              UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
              ... ...

    lo        Link encap:Local Loopback
              inet addr:127.0.0.1  Mask:255.0.0.0
              UP LOOPBACK RUNNING  MTU:16436  Metric:1
              ... ...

    wlan0     Link encap:Ethernet  HWaddr B0:48:7A:3B:EF:64
              UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
              ... ...

从路由表中也可以看出配置的 IP alias 生效了：

    # route -n
    Kernel IP routing table
    Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
    0.0.0.0         10.10.15.254    0.0.0.0         UG    0      0        0 br-lan
    10.10.15.0      0.0.0.0         255.255.255.0   U     0      0        0 br-lan
    192.168.1.0     0.0.0.0         255.255.255.0   U     0      0        0 br-lan

这篇文章 [may be a bug: alias interface do not show on ifconfig][2] 提到 `ip addr`

[2]: https://forum.openwrt.org/viewtopic.php?id=37842

想到 `sencondary ip address` 使用 `ifconfig` 是看不到的，安装 `opkg install ip` 软件包

    # ip a
    1: lo: <LOOPBACK,UP,LOWER_UP> mtu 16436 qdisc noqueue state UNKNOWN
        link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
        inet 127.0.0.1/8 scope host lo
    2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast master br-lan state UP qlen 1000
        link/ether b0:48:7a:3b:ef:64 brd ff:ff:ff:ff:ff:ff
    10: br-lan: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP
        link/ether b0:48:7a:3b:ef:64 brd ff:ff:ff:ff:ff:ff
        inet 10.10.15.38/24 brd 10.10.15.255 scope global br-lan
        inet 192.168.1.12/24 brd 192.168.1.255 scope global br-lan
        inet 10.10.15.187/24 brd 10.10.15.255 scope global secondary br-lan
    11: wlan0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq master br-lan state UP qlen 32
        link/ether b0:48:7a:3b:ef:64 brd ff:ff:ff:ff:ff:ff

果然 openwrt 的 alias 其实是 **辅助 ip 地址 (secondary ip address)**

而不是 **ip 别名 (ip alias)** 两者区别可以参考下面这篇文章：

[从 ip addr add 和 ifconfig 的区别看 linux 网卡 ip 地址的结构][2]

[2]: http://blog.csdn.net/dog250/article/details/5303542

## bridge AP vs 虚化化网桥

从路由器的视角来看 bridged AP 有点类似 linux 中虚拟化的桥接 ：

    # brctl show
    bridge name     bridge id               STP enabled     interfaces
    br-lan          8000.b0487a3aee72       no              eth0
                                                            wlan0

Linux 宿主机的桥接设备还是需要配置可用的 IP  不然下面的虚拟机网络就没法用了。
对于 wr703n 路由器，只有一个 `wan / lan` 自适应网口 openwrt 中即使 `br-lan` 网桥地址设置错误
bridged AP 模式可以正常用，只是无法连接管理地址，登录 openwrt 进行管理而已

看得出来 **接入点模式** 真的很无视路由器的存在哦 -_-#





<br/>

本文标题 | [{{ page.title }}]({{ page.url }})
-------- |:--------
原始链接 | <{{ site.url }}{{ page.url }}>
