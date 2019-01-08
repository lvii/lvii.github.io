---
layout: post
title: "通过 ifcfg-eth0 管理 /etc/resolv.conf"
category: system
tags: [network]
---

kickstart 装机 post-inst 阶段配置网卡，还要一同配置 `/etc/resolv.conf`

后来用户要增加新的 `search DOMAIN` 或修改 DNS IP

这样要修改 **两处** 配置信息，有点麻烦。

干脆把 `/etc/resolv.conf` 配置并入 `ifcfg-eth0` ：

`ifcfg-eth0` | `/etc/resolv.conf`
:------------ | :-----------------
`PEERDNS=yes` | 　
`DNS1=1.1.1.1` | `nameserver 1.1.1.1`
`DNS2=8.8.8.8` | `nameserver 8.8.8.8`
`DOMAIN='example.com example-inc.com'` | `search example.com example-inc.com`

- `PEERDNS=yes` 选项配置 `ifcfg-eth0` 的 `DNS1`，`DNS2` 为 `/etc/resolv.conf` 的 `nameserver`
- `DOMAIN='example.com ...'` 选项配置 `/etc/resolv.conf` 的默认搜索域 `search` 字段

官方文档对 `PEERDNS=yes` 的介绍：

<https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/6/html/deployment_guide/s1-networkscripts-interfaces>

> `PEERDNS=answer`
>
> where `answer` is one of the following:
>
> - `yes` — Modify `/etc/resolv.conf` if the DNS directive is set, **if using DHCP**, or if using Microsoft's RFC 1877 IPCP extensions with PPP. In all cases `yes` is the **default**.
> - `no` — Do not modify `/etc/resolv.conf`.
>
> `DNS{1,2}=address`
>
> where `address` is a name server address to be placed in `/etc/resolv.conf` provided that the `PEERDNS` directive is **not** set to `no`.

[How can I add additional search domains to the resolv.conf created by dhclient in CentOS](https://superuser.com/questions/110808/how-can-i-add-additional-search-domains-to-the-resolv-conf-created-by-dhclient-i/466912#466912)

多个搜索域一定要用 **"引号"** 包裹，不然无法更新 `/etc/resolv.conf` 的 `search` 字段。

    # cat /etc/sysconfig/network-scripts/ifcfg-eth0
    DEVICE=eth0
    NAME=eth0
    TYPE=Ethernet
    ONBOOT=yes
    BOOTPROTO=none
    HWADDR=90:bd:11:1a:2a:b2
    IPADDR=10.16.5.1
    PREFIX=24
    GATEWAY=10.16.5.254
    # config /etc/resolv.conf
    PEERDNS=yes
    DNS1=1.1.1.1
    DNS2=8.8.8.8
    DOMAIN='example.com example-inc.com'    # <-- 多个搜索域要用 "引号" 包裹

    # service network restart
    Restarting network (via systemctl):                        [  OK  ]

    # cat /etc/resolv.conf
    # nameserver 1.1.1.1
    # nameserver 8.8.8.8
    # search example.com
    nameserver 1.1.1.1
    nameserver 8.8.8.8
    search example.com example-inc.com

修改 `/etc/resolv.conf` 文件是 `/etc/sysconfig/network-scripts/ifup-post` 脚本：

    # fgrep -w DOMAIN /etc/sysconfig/network-scripts/*
    /etc/sysconfig/network-scripts/ifup-post:    if [ -n "${DOMAIN}" ] && ! grep -q "^search.*${DOMAIN}.*$" /etc/resolv.conf ||
    /etc/sysconfig/network-scripts/ifup-post:                        if [ -n "${DOMAIN}" ]; then
    /etc/sysconfig/network-scripts/ifup-post:            if [ -n "${DOMAIN}" ]; then
    /etc/sysconfig/network-scripts/ifup-post:                echo "search ${DOMAIN}${search_str}" >> "${tmp_file}"

修改 `search` 字段对应的 shell 代码：

``` sh
while read line; do
    case ${line} in

        # Skip nameserver entries when at least one DNS option was given
        # (at this stage we know that we have to update all the nameserver
        # enries anyway -- see below), or copy them if we are changing just
        # the 'search' field in /etc/resolv.conf:
        nameserver*)
            if [[ "${grep_regexp}" != ".*" ]]; then
                continue
            else
                echo "${line}" >> "${tmp_file}"
            fi
            ;;

        domain* | search*)
            if [ -n "${DOMAIN}" ]; then
                read search value < <(echo ${line})
                search_str+=" ${value}"
            else
                echo "${line}" >> "${tmp_file}"
            fi
            ;;

        # Keep the rest of the /etc/resolv.conf as it was:
        *)
            echo "${line}" >> "${tmp_file}"
            ;;
    esac
done < /etc/resolv.conf

# Insert the domain into 'search' field:
if [ -n "${DOMAIN}" ]; then
    echo "search ${DOMAIN}${search_str}" >> "${tmp_file}"
fi
```

<br/>

本文标题 | [{{ page.title }}]({{ page.url }})
-------- |:--------
原始链接 | <{{ site.url }}{{ page.url }}>
