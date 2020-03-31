---
layout: post
title: "HP iLO 登录用户名与 BMC 用户名不一致导致 ipmitool 无法修改用户名"
category: system
tags: [ipmitool]
---

# WHAT

一台 HP 服务器更换主板后，使用 `ipmitool` 重新初始化 BMC 用户时，修改用户名失败：

    # ipmitool user enable 3

    # ipmitool user set name 3 root
    Set User Name command failed (user 3, name root): Unspecified error

    # bmc-config -c -e User3:Username="root"
    ERROR: Failed to commit `User3:Username'

# HOW

修改用户密码操作是正常的，尝试换一个用户名测试了一下竟然可以：

    # bmc-config -c -e User${uid}:Username="roott"

    # for i in {1..3}; do bmc-config -o -e User${i}:Username; done
    Section User1
            ## Give Username
            ## Username                                   NULL
    EndSection
    Section User2
            ## Give Username
            Username                                      admin
    EndSection
    Section User3
            ## Give Username
            Username                                      roott
    EndSection

随后也可以将用户名修改回 `root` 用户，这操作有点懵：

    # bmc-config -c -e User${uid}:Username="root"

    # for i in {1..3}; do bmc-config -o -e User${i}:Username; done
    Section User1
            ## Give Username
            ## Username                                   NULL
    EndSection
    Section User2
            ## Give Username
            Username                                      admin
    EndSection
    Section User3
            ## Give Username
            Username                                      root
    EndSection

于是进入 iLO web 后台对应的用户管理查看用户：

![img](https://i.imgur.com/dqfABVM.png)

发现 iLO 登录 Web 后台用户名和 BMC 管理卡用户名是可以分别指定的。

开始是因为 `admin` 登录用户占用了 `root` 用户名，导致无法修改成 `root` 用户。

于是删掉上面对应关系混乱的用户，重新初始化了一下，解决问题。

<br/>

本文标题 | [{{ page.title }}]({{ page.url }})
-------- |:--------
原始链接 | <{{ site.url }}{{ page.url }}>