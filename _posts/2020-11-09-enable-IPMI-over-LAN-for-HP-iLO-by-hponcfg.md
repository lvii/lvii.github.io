---
layout: post
title: "通过 hponcfg 启用 HP iLO 的 IPMI over LAN 特性"
category: server
tags: [ipmi,iLO]
---

# WHAT

有台 HP 机器发现无法使用 `ipmitool` 重启：

    # ipmitool -I lanplus -U root -f ~/.oob -H 10.13.19.12 power status
    Error: Unable to establish IPMI v2 / RMCP+ session

但是 iLO 的 web 都是正常的，开始以为是用户名导致的问题，修改完用户名发现还是不行

# WHY

排查了一番发现是因为 HP iLO 的 **IPMI over LAN** 特性没有开启

在 iLO web 后台 **Administration** 页面下的 **Access Settings** 目录可以找到 **IPMI/DCMI over LAN Access** 开关

![img](https://i.imgur.com/lpXFR6L.png)

**NOTE** ：开关 **IPMI/DCMI over LAN Access** 设置选项 HP iLO 会 **重启**

# HOW

如果是要批量修改，可以使用 `hponcfg` 工具操作：

    # rpm -qf `which hponcfg`
    hponcfg-5.4.0-0.x86_64

先使用 `hponcfg` 查看 iLO **全局** 配置，用来生成后面的修改配置文件：

    echo '<RIBCL VERSION="2.0">
      <LOGIN USER_LOGIN="Administrator" PASSWORD="password">
        <RIB_INFO MODE="read">
          <GET_GLOBAL_SETTINGS />
        </RIB_INFO>
      </LOGIN>
    </RIBCL>' | hponcfg -i

    # echo '<RIBCL VERSION="2.0">
    >   <LOGIN USER_LOGIN="Administrator" PASSWORD="password">
    >     <RIB_INFO MODE="read">
    >       <GET_GLOBAL_SETTINGS />
    >     </RIB_INFO>
    >   </LOGIN>
    > </RIBCL>'|hponcfg -i
    HP Lights-Out Online Configuration utility
    Version 5.4.0 Date 8/6/2018 (c) 2005,2018 Hewlett Packard Enterprise Development LP
    Firmware Revision = 2.61 Device type = iLO 4 Driver name = hpilo
    <GET_GLOBAL_SETTINGS>
        <SESSION_TIMEOUT VALUE="30"/>
        <ILO_FUNCT_ENABLED VALUE="Y"/>
        <F8_PROMPT_ENABLED VALUE="Y"/>
        <F8_LOGIN_REQUIRED VALUE="N"/>
        <HTTPS_PORT VALUE="443"/>
        <HTTP_PORT VALUE="80"/>
        <REMOTE_CONSOLE_PORT VALUE="17990"/>
        <VIRTUAL_MEDIA_PORT VALUE="17988"/>
        <SNMP_ACCESS_ENABLED VALUE="Y"/>
        <SNMP_PORT VALUE="161"/>
        <SNMP_TRAP_PORT VALUE="162"/>
        <SSH_PORT VALUE="22"/>
        <SSH_STATUS VALUE="Y"/>
        <SERIAL_CLI_STATUS VALUE="Enabled-Authentication Required"/>
        <SERIAL_CLI_SPEED VALUE="9600"/>
        <VSP_LOG_ENABLE VALUE="N"/>
        <MIN_PASSWORD VALUE="5"/>
        <AUTHENTICATION_FAILURE_LOGGING VALUE="Enabled-every 3rd failure"/>
        <AUTHENTICATION_FAILURE_DELAY_SECS VALUE="10"/>
        <AUTHENTICATION_FAILURES_BEFORE_DELAY VALUE="1"/>
        <LOCK_CONFIGURATION VALUE="N"/>
        <RBSU_POST_IP VALUE="Y"/>
        <ENFORCE_AES VALUE="N"/>
        <IPMI_DCMI_OVER_LAN_ENABLED VALUE="N"/>             <-- IPMI over LAN
        <REMOTE_SYSLOG_ENABLE VALUE="N"/>
        <REMOTE_SYSLOG_PORT VALUE="514"/>
        <REMOTE_SYSLOG_SERVER_ADDRESS VALUE=""/>
        <ALERTMAIL_ENABLE VALUE="N"/>
        <ALERTMAIL_EMAIL_ADDRESS VALUE=""/>
        <ALERTMAIL_SENDER_DOMAIN VALUE=""/>
        <ALERTMAIL_SMTP_PORT VALUE="25"/>
        <ALERTMAIL_SMTP_SERVER VALUE=""/>
        <PROPAGATE_TIME_TO_HOST VALUE="N"/>
        <IPMI_DCMI_OVER_LAN_PORT VALUE="623"/>
    </GET_GLOBAL_SETTINGS>
    Script succeeded

使用 `hponcfg` 修改 `IPMI_DCMI_OVER_LAN_ENABLED` 配置：

    echo '<RIBCL VERSION="2.0">
      <LOGIN USER_LOGIN="Administrator" PASSWORD="password">
        <RIB_INFO MODE="write">
          <MOD_GLOBAL_SETTINGS>
            <IPMI_DCMI_OVER_LAN_ENABLED VALUE="Y"/>
          </MOD_GLOBAL_SETTINGS>
        </RIB_INFO>
      </LOGIN>
    </RIBCL>'|hponcfg -i

    + echo '<RIBCL VERSION="2.0">
      <LOGIN USER_LOGIN="Administrator" PASSWORD="password">
        <RIB_INFO MODE="write">
          <MOD_GLOBAL_SETTINGS>
            <IPMI_DCMI_OVER_LAN_ENABLED VALUE="Y"/>
          </MOD_GLOBAL_SETTINGS>
        </RIB_INFO>
      </LOGIN>
    </RIBCL>'
    + hponcfg -i
    HP Lights-Out Online Configuration utility
    Version 5.4.0 Date 8/6/2018 (c) 2005,2018 Hewlett Packard Enterprise Development LP
    Firmware Revision = 2.61 Device type = iLO 4 Driver name = hpilo
    <INFORM>Integrated Lights-Out will reset at the end of the script.</INFORM>

    Please wait while the firmware is reset. This might take a minute
    Script succeeded

使用 `hponcfg` 修改 iLO 的 **IPMI over LAN** 也会 **重启**

# reference

[Automate disabling of IPMI over LAN access on HPE iLO 2018-11-14](https://rudimartinsen.com/2018/11/14/automate-disabling-of-ipmi-over-lan-access-on-hpe-ilo/)

[HPE Integrity Superdome X Servers & HPE Superdome Flex - Security Vulnerability CVE-2013-4786](https://support.hpe.com/hpesc/public/docDisplay?docId=emr_na-a00026813en_us)