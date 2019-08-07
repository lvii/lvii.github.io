---
layout: post
title: "iPXE 引导 ubuntu 并使用 preseed 自动安装系统"
category: system
tags: [ubuntu]
---

# WHY

厂里有项目要用 ubuntu 系统，折腾一下 ubuntu 的 preseed 自动装机

# HOW

## iPXE boot

挂载 ISO 提供 PXE 所需内核引导文件：

    mount /home/iso/ubuntu-16.04.6-server-amd64.iso /var/www/html/iso/ubuntu

添加 ubuntu 对应的 iPXE 引导菜单：

    set iso_u http://10.20.30.1/iso/ubuntu

    :ubuntu
    initrd ${iso_u}/install/netboot/ubuntu-installer/amd64/initrd.gz ||
    kernel ${iso_u}/install/netboot/ubuntu-installer/amd64/linux BOOTIF=01-${net0/mac:hexhyp} interface=auto auto=true priority=critical url=http://${next-server}/ubuntu.cfg DEBCONF_DEBUG=5 --- net.ifnames=0 ipv6.disable=1 ||

内核参数 | 作用
:------- | :---
`interface=auto` | 跳过手动选择引导网卡界面
`auto=true priority=critical url=...` | 根据 `url=` 指定的 preseed 配置自动安装系统
`DEBCONF_DEBUG=5` | 控制台 console `Alt + F4` 输出更详细日志
`--- net.ifnames=0 ipv6.disable=1` | `---` 之后的参数，装机完成后保留（**实测无效**）

## preseed

preseed 文件要实现真正自动化，需要不少 hack 和 CentOS 的 kickstart 相比，语法和文档差太多。。。


### network

配置静态网络在加载好 LiveOS 环境 DHCP 完成后，需要 **重启网络** 重新加载 preseed 配置：

    d-i preseed/early_command string kill-all-dhcp; netcfg

[Appendix B. Automating the installation using preseeding B.4. Contents of the preconfiguration file](https://www.debian.org/releases/buster/amd64/apbs04.en.html)

> Although preseeding the network configuration is normally **NOT** possible when using network preseeding (using “preseed/url”), you can use the following hack to work around that, for example if you'd like to set a **static address** for the network interface. The hack is to **force the network configuration to run again after the preconfiguration file has been loaded by creating a “preseed/run” script containing the following commands: `kill-all-dhcp; netcfg`

而且 iPXE 内核引导参数需要加上 `priority=critical` 不然还会弹框手动确认

### repo

默认安装会下载 **公网** 的 security 源，超时时间很长，安装界面卡很久，像 **死机** 一般：

![img_ubuntu_preseed_download_security_repo_timeout](https://i.imgur.com/uZC8iEC.png)

将 security 源也指向 ISO 对应的源：

    d-i apt-setup/security_host string 10.20.30.1

### postfix

安装额外软件包引入 `postfix` 依赖，配置 postfix 界面会中断 preseed 自动安装：

![img_ubuntu_preseed_config_postfix](https://i.imgur.com/YZE5XCZ.png)

再补一刀：

    postfix postfix/main_mailer_type select No configuration

### late_command

`late_command` 里面如果有 `echo ... > redirect_file` **重定向** 操作，直接在 `in-target` 里用无效

需要用 `sh -c 'command'` 包装一下：


    d-i preseed/late_command string \
    in-target sh -c 'mkdir -pv --mode=0700 /root/.ssh'; \
    test -d /target/root/.ssh || { mkdir -pv /target/root/.ssh; chmod -v 700 /target/root/.ssh; } && ls -la /target/root|grep ssh; \
    in-target sh -c 'echo -e "ssh-rsa AAAAB..." > /root/.ssh/authorized_keys && chmod -v 0600 /root/.ssh/authorized_keys'; \

再就是难用的分区配置。。。

### example preseed config

最后汇总的 preseed 配置如下：

    d-i debian-installer/locale string en_US

    d-i console-setup/ask_detect boolean false
    d-i keyboard-configuration/xkb-keymap select us
    d-i keyboard-configuration/layoutcode string us

    d-i clock-setup/utc boolean false
    d-i clock-setup/ntp boolean false
    d-i time/zone string Asia/Shanghai

    d-i hw-detect/load_firmware boolean true

    #    _  ____________  _________  _  _______________
    #   / |/ / __/_  __/ / ___/ __ \/ |/ / __/  _/ ___/
    #  /    / _/  / /   / /__/ /_/ /    / _/_/ // (_ /
    # /_/|_/___/ /_/    \___/\____/_/|_/_/ /___/\___/
    #

    # NOTE: not work without iPXE boot option: interface=auto
    d-i netcfg/choose_interface select auto

    # Static Network
    # NOTE: not work without iPXE boot option: auto=true priority=critical
    d-i preseed/early_command string kill-all-dhcp; netcfg

    d-i netcfg/disable_autoconfig boolean true
    d-i netcfg/disable_dhcp boolean true

    d-i netcfg/get_ipaddress string 10.20.30.3
    d-i netcfg/get_netmask string 255.255.255.0
    d-i netcfg/get_gateway string 10.20.30.254
    d-i netcfg/get_hostname string ubuntu

    d-i netcfg/get_nameservers string 1.1.1.1 8.8.8.8
    d-i netcfg/get_domain string example.com

    d-i netcfg/confirm_static boolean true
    d-i netcfg/dhcp_options select Configure network manually

    # DHCP Network
    # d-i netcfg/get_hostname string ubuntu
    # d-i netcfg/get_domain string ""
    # d-i netcfg/hostname string ubuntu


    #    __  __________  ___  ____  ___
    #   /  |/  /  _/ _ \/ _ \/ __ \/ _ \
    #  / /|_/ // // , _/ , _/ /_/ / , _/
    # /_/  /_/___/_/|_/_/|_|\____/_/|_|
    #
    d-i live-installer/net-image string http://10.20.30.1/iso/ubuntu/install/filesystem.squashfs

    # d-i debian-installer/allow_unauthenticated string true # Allow local source
    d-i mirror/country string manual
    d-i mirror/http/hostname string 10.20.30.1
    d-i mirror/http/directory string /iso/ubuntu/
    d-i mirror/http/proxy string
    d-i mirror/http/mirror select 10.20.30.1

    # NOTE: this would skip download repo metadata frome 'security.ubuntu.com'
    d-i apt-setup/use_mirror boolean false
    d-i apt-setup/security_host string 10.20.30.1


    #    ___  ___   ___  ________________________  _  __
    #   / _ \/ _ | / _ \/_  __/  _/_  __/  _/ __ \/ |/ /
    #  / ___/ __ |/ , _/ / / _/ /  / / _/ // /_/ /    /
    # /_/  /_/ |_/_/|_| /_/ /___/ /_/ /___/\____/_/|_/
    #
    d-i partman-auto/disk string /dev/sda
    d-i partman-auto/method string regular
    d-i partman-auto/purge_lvm_from_device boolean true
    d-i partman-lvm/device_remove_lvm boolean true
    d-i partman-lvm/confirm boolean true
    d-i partman-md/device_remove_md boolean true
    d-i partman-auto/choose_recipe select atomic

    # d-i partman-auto/expert_recipe string                         \
    #       boot-root ::                                            \
    #               1 1 2 free                                      \
    #               $gptonly{ }                                     \
    #               $primary{ }                                     \
    #               $bios_boot{ }                                   \
    #               method{ biosgrub }                              \
    #               .                                               \
    #               300 600 900 xfs                                 \
    #                       $primary{ } $bootable{ }                \
    #                       method{ format } format{ }              \
    #                       use_filesystem{ } filesystem{ xfs }     \
    #                       mountpoint{ /boot }                     \
    #               .                                               \
    #               6000 60000 -1 xfs                               \
    #                       $primary{ }                             \
    #                       method{ format } format{ }              \
    #                       use_filesystem{ } filesystem{ xfs }     \
    #                       mountpoint{ / }                         \
    #               .                                               \
    #               3027 4096 5120 linux-swap                       \
    #                       $primary{ }                             \
    #                       method{ swap } format{ }                \
    #               .

    # d-i partman-basicfilesystems/no_swap boolean false

    d-i partman-partitioning/confirm_write_new_label boolean true
    d-i partman/choose_partition select finish
    d-i partman/confirm boolean true
    d-i partman/confirm_nooverwrite boolean true


    #    ___  ___  _______ _____  _________
    #   / _ \/ _ |/ ___/ //_/ _ |/ ___/ __/
    #  / ___/ __ / /__/ ,< / __ / (_ / _/
    # /_/  /_/ |_\___/_/|_/_/ |_\___/___/
    #
    # python3-jinja2 python3-lxml python3-yaml python3-paramiko python3-pexpect python3-prettytable python3-netaddr (ipcalc)

    # tasksel tasksel/first multiselect standard, server, openssh-server
    # d-i pkgsel/include string bc tmux lsscsi smartmontools nmap ntpdate openipmi freeipmi-tools

    tasksel tasksel/first multiselect standard, openssh-server
    d-i pkgsel/exclude string nano screen plymouth
    d-i pkgsel/include string bc tmux lsscsi smartmontools nmap ntpdate openipmi freeipmi-tools
    # d-i pkgsel/include string bc vim tmux gawk nmap rename ethtool ifenslave vlan ntpdate openipmi freeipmi-tools

    # d-i pkgsel/language-packs multiselect en
    d-i pkgsel/install-language-support boolean false

    # NOTE: skip "updating the list of available packages"
    d-i pkgsel/upgrade select none
    d-i pkgsel/update-policy select none

    # d-i pkgsel/upgrade select full-upgrade
    # d-i pkgsel/update-policy select unattended-upgrades

    postfix postfix/main_mailer_type select No configuration


    #   __  _____________
    #  / / / / __/ __/ _ \
    # / /_/ /\ \/ _// , _/
    # \____/___/___/_/|_|
    #
    # python -c 'import crypt,getpass;pw=getpass.getpass();print(crypt.crypt(pw) if (pw==getpass.getpass("Confirm: ")) else exit())'
    d-i passwd/root-login boolean true
    d-i passwd/root-password-crypted password ...

    d-i passwd/username string worker
    d-i passwd/user-fullname string worker
    d-i passwd/user-password-crypted password ...
    d-i passwd/user-default-groups string sudo

    # d-i passwd/user-password password insecure
    # d-i passwd/user-password-again password insecure
    # d-i user-setup/allow-password-weak boolean true

    d-i user-setup/encrypt-home boolean false


    #   ________  __  _____
    #  / ___/ _ \/ / / / _ )
    # / (_ / , _/ /_/ / _  |
    # \___/_/|_|\____/____/
    #
    d-i grub-installer/only_debian boolean true
    d-i debian-installer/add-kernel-opts string net.ifnames=0 ipv6.disable=1 cgroup_enable=memory swapaccount=1

    # NOTE: remove 'quiet splash' from grub config
    d-i debian-installer/quiet	boolean false
    d-i debian-installer/splash	boolean false


    #    ___  ____  __________
    #   / _ \/ __ \/ __/_  __/
    #  / ___/ /_/ /\ \  / /
    # /_/   \____/___/ /_/
    #
    d-i preseed/late_command string \
    in-target sh -c 'mkdir -pv --mode=0700 /root/.ssh'; \
    test -d /target/root/.ssh || { mkdir -pv /target/root/.ssh; chmod -v 700 /target/root/.ssh; } && ls -la /target/root|grep ssh; \
    in-target sh -c 'echo -e "ssh-rsa AAAAB..." > /root/.ssh/authorized_keys && chmod -v 0600 /root/.ssh/authorized_keys'; \
    in-target sed -i -e 's/^\(PasswordAuthentication\).*/\1 yes/g' -e 's/^\(PermitRootLogin\).*/\1 yes/g' /etc/ssh/sshd_config; \
    in-target sed -i -e '/^GRUB_HIDDEN_TIMEOUT=/d' -e 's/^\(GRUB_HIDDEN_TIMEOUT_QUIET\)=true/\1=false/' /etc/default/grub; \
    in-target update-grub

    d-i finish-install/reboot_in_progress

<br/>

本文标题 | [{{ page.title }}]({{ page.url }})
-------- |:--------
原始链接 | <{{ site.url }}{{ page.url }}>
