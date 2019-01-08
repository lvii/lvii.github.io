---
layout: post
title: "浪潮服务器 BMC 用户配置问题导致 ipmipower 出现 username invalid 报错"
category: system
tags: [freeipmi, ipmitool]
---

# WHAT

厂里的 **浪潮 (Inspur) 服务器** 使用 `ipmipower` 远程管理电源，提示 `username invalid` 错误：

    # ipmipower -D LAN_2_0 --session-timeout=1000 -u root -p $(cat ~/oob) -h 10.30.2.17 -s
    10.30.2.17: username invalid

奇葩的是 `ipmitool` 却能正常使用：

    # ipmitool -I lanplus -U root -P $(cat ~/oob) -H 10.30.2.17 power status
    Chassis Power is on

# WHY

通过 `bmc-config` ( `freeipmi` 软件包 ) 查看 BMC 用户信息，发现和 `ipmitool` 的结果不同：

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
            ## Username                                   <username-not-set-yet>
    EndSection

    # ipmitool user list 1
    ID  Name             Callin  Link Auth  IPMI Msg   Channel Priv Limit
    1   root             true    true       true       ADMINISTRATOR
    2   admin            false   false      true       ADMINISTRATOR

# HOW

尝试用 `bmc-config` 修改 `User1` 对应的用户名，结果失败：

    # bmc-config -c -e User1:Username="root"
    Invalid value 'root' for key 'Username' in section 'User1'

但是 `ipmitool` 是可以修改 User ID 为 `1` 的用户名：

    # ipmitool user set name 1 NULL

    # ipmitool user list 1
    ID  Name             Callin  Link Auth  IPMI Msg   Channel Priv Limit
    1   NULL             true    true       true       ADMINISTRATOR
    2   admin            true    true       true       ADMINISTRATOR

## wrong way

之后尝试修改 User ID 为 `2` 的 `admin` 用户名为 `root` ：

    # ipmitool user list 1
    ID  Name             Callin  Link Auth  IPMI Msg   Channel Priv Limit
    1   NULL             true    true       true       ADMINISTRATOR
    2   root             true    true       true       ADMINISTRATOR

    # bmc-config -o -e User$(ipmitool user list 1|awk '/\<root\>/{print $1}'):Username
    Section User2
            ## Give Username
            Username                                      root
    EndSection

虽然能修改用户名，但是 `ipmipower` 远程测试，还是提示 `username invalid` 报错：

    # ipmipower -D LAN_2_0 --session-timeout=1000 -u root -p $(cat ~/oob) -h 10.30.2.17 -s
    10.30.2.17: username invalid

    # ipmitool -I lanplus -U root -P $(cat ~/oob) -H 10.30.2.17 power status
    Chassis Power is on

## right way

`bmc-config` 用户 `User1` 配置详情，仍然无法看到使用 `ipmitool` 修改过的用户名：

    # bmc-config -o -S User1
    #
    # Section UserX Comments
    #
    # In the following User sections, users should configure usernames, passwords,
    # and access rights for IPMI over LAN communication. Usernames can be set to any
    # string with the exception of User1, which is a fixed to the "anonymous"
    # username in IPMI.
    #
    # For IPMI over LAN access for a username, set "Enable_User" to "Yes",
    # "Lan_Enable_IPMI_Msgs" to "Yes", and "Lan_Privilege_Limit" to a privilege
    # level. The privilege level is used to limit various IPMI operations for
    # individual usernames. It is recommened that atleast one username be created
    # with a privilege limit "Administrator", so all system functions are available
    # to atleast one username via IPMI over LAN. For security reasons, we recommend
    # not enabling the "anonymous" User1. For most users, "Lan_Session_Limit" can be
    # set to 0 (or ignored) to support an unlimited number of simultaneous IPMI over
    # LAN sessions.
    #
    # If your system supports IPMI 2.0 and Serial-over-LAN (SOL),
    # a"SOL_Payload_Access" field may be listed below. Set the "SOL_Payload_Access"
    # field to "Yes" or "No" to enable or disable this username's ability to access
    # SOL.
    #
    # Please do not forget to uncomment those fields, such as "Password", that may
    # be commented out during the checkout.
    #
    # Some motherboards may require a "Username" to be configured prior to other
    # fields being read/written. If this is the case, those fields will be set to
    # <username-not-set-yet>.
    #
    Section User1
            ## Give Username
            ## Username                                   NULL
            ## Give password or blank to clear. MAX 16 chars (20 chars if IPMI 2.0 supported).
            ## Password
            ## Possible values: Yes/No or blank to not set
            Enable_User                                   Yes
            ## Possible values: Yes/No
            Lan_Enable_IPMI_Msgs                          Yes
            ## Possible values: Yes/No
            Lan_Enable_Link_Auth                          Yes
            ## Possible values: Yes/No
            Lan_Enable_Restricted_to_Callback             No
            ## Possible values: Callback/User/Operator/Administrator/OEM_Proprietary/No_Access
            Lan_Privilege_Limit                           Administrator
            ## Possible values: 0-17, 0 is unlimited; May be reset to 0 if not specified
            ## Lan_Session_Limit
            ## Possible values: Yes/No
            SOL_Payload_Access                            Yes
    EndSection

后来联系厂家，厂家工程师建议使用 `ipmitool` 或是 **新建用户** 试试看。

测试 **新建用户** ：

默认 `User3` 用户 **未启用** ：`Enable_User No`

    # bmc-config -o -S User3
    Section User3
            ## Give Username
            ## Username                                   <username-not-set-yet>
            ## Give password or blank to clear. MAX 16 chars (20 chars if IPMI 2.0 supported).
            ## Password
            ## Possible values: Yes/No or blank to not set
            Enable_User                                   No
            ## Possible values: Yes/No
            Lan_Enable_IPMI_Msgs                          No
            ## Possible values: Yes/No
            Lan_Enable_Link_Auth                          No
            ## Possible values: Yes/No
            Lan_Enable_Restricted_to_Callback             No
            ## Possible values: Callback/User/Operator/Administrator/OEM_Proprietary/No_Access
            Lan_Privilege_Limit                           No_Access
            ## Possible values: 0-17, 0 is unlimited; May be reset to 0 if not specified
            ## Lan_Session_Limit
            ## Possible values: Yes/No
            ## SOL_Payload_Access                         <username-not-set-yet>
    EndSection

配置并启用 `User3` 用户：

    uid=3
    bmc-config -c -e User${uid}:Username="root"
    bmc-config -c -e User${uid}:Password="...."
    bmc-config -c -e User${uid}:Enable_User=Yes
    bmc-config -c -e User${uid}:Lan_Enable_IPMI_Msgs=Yes
    bmc-config -c -e User${uid}:Lan_Enable_Link_Auth=Yes
    bmc-config -c -e User${uid}:Lan_Privilege_Limit=Administrator
    bmc-config -c -e User${uid}:SOL_Payload_Access=Yes

配置完成后的 `User3` 用户：

    # bmc-config -o -S User3
    Section User3
            ## Give Username
            Username                                      root
            ## Give password or blank to clear. MAX 16 chars (20 chars if IPMI 2.0 supported).
            ## Password
            ## Possible values: Yes/No or blank to not set
            Enable_User                                   Yes
            ## Possible values: Yes/No
            Lan_Enable_IPMI_Msgs                          Yes
            ## Possible values: Yes/No
            Lan_Enable_Link_Auth                          Yes
            ## Possible values: Yes/No
            Lan_Enable_Restricted_to_Callback             No
            ## Possible values: Callback/User/Operator/Administrator/OEM_Proprietary/No_Access
            Lan_Privilege_Limit                           Administrator
            ## Possible values: 0-17, 0 is unlimited; May be reset to 0 if not specified
            ## Lan_Session_Limit
            ## Possible values: Yes/No
            SOL_Payload_Access                            Yes
    EndSection

所有启用的 BMC 用户：

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

    # ipmitool user list 1
    ID  Name             Callin  Link Auth  IPMI Msg   Channel Priv Limit
    1   NULL             true    true       true       ADMINISTRATOR
    2   admin            true    true       true       ADMINISTRATOR
    3   root             true    true       true       ADMINISTRATOR

然后再次测试发现 **所有用户** 均都正常了：

    # ipmipower -D LAN_2_0 --session-timeout=1000 -u root -p $(cat ~/oob) -h 10.30.2.17 -s
    10.30.2.17: on

    # ipmipower -D LAN_2_0 --session-timeout=1000 -u NULL -p $(cat ~/oob) -h 10.30.2.17 -s
    10.30.2.17: on

    # ipmipower -D LAN_2_0 --session-timeout=1000 -u admin -p $(cat ~/oob) -h 10.30.2.17 -s
    10.30.2.17: on

后面排查一下其他厂商的机器，发现 `User1` 用户都是 **不可配置** 的，配置 `User2` 是可行的

但是 **浪潮** 机器配置 `User2` 依然无效，需要 **新建用户** 才能工作。。。

<br/>

本文标题 | [{{ page.title }}]({{ page.url }})
-------- |:--------
原始链接 | <{{ site.url }}{{ page.url }}>
