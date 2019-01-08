---
layout: post
title: "从 Fedora 28 升级更新至 Fedora 29"
category: system
tags: [fedora]
date: 2018-10-10 12:00:00 +08:00
---

Fedora 29 Beta 早就发布了，前两天也已经 Final Freeze 如果不跳票的话，这个月就发布了，不等了：

<https://fedoraproject.org/wiki/DNF_system_upgrade>

    # dnf install dnf-plugin-system-upgrade
    Package python3-dnf-plugin-system-upgrade-2.0.5-3.fc28.noarch is already installed.
    Dependencies resolved.
    Nothing to do.
    Complete!

导入 Fedora 29 的 GPG KEY 校验安装包：

    # rpm -qf /etc/pki/rpm-gpg/RPM-GPG-KEY-fedora-29-primary
    fedora-gpg-keys-28-5.noarch

    # rpm -q gpg-pubkey --qf '%{NAME}-%{VERSION}-%{RELEASE}\t%{SUMMARY}\n'
    gpg-pubkey-f5282ee4-58ac92a3    gpg(Fedora 27 (27) <fedora-27@fedoraproject.org>)
    gpg-pubkey-9db62fb1-59920156    gpg(Fedora 28 (28) <fedora-28@fedoraproject.org>)

    # rpm --import /etc/pki/rpm-gpg/RPM-GPG-KEY-fedora-29-primary

    # rpm -q gpg-pubkey --qf '%{NAME}-%{VERSION}-%{RELEASE}\t%{SUMMARY}\n'
    gpg-pubkey-f5282ee4-58ac92a3    gpg(Fedora 27 (27) <fedora-27@fedoraproject.org>)
    gpg-pubkey-9db62fb1-59920156    gpg(Fedora 28 (28) <fedora-28@fedoraproject.org>)
    gpg-pubkey-429476b4-5a886537    gpg(Fedora 29 (29) <fedora-29@fedoraproject.org>)


更新 `/etc/dnf/dnf.conf` 忽略 weak 依赖、避免引入 `i686` 软件包：

    # echo -e 'install_weak_deps=False\nexclude=*.i686\n# tsflags=nodocs' >> /etc/dnf/dnf.conf

    # cat /etc/dnf/dnf.conf
    [main]
    gpgcheck=1
    installonly_limit=2
    clean_requirements_on_remove=True
    install_weak_deps=False
    exclude=*.i686
    # tsflags=nodocs

下载更新所需 rpm 包：

    # dnf system-upgrade download --refresh --releasever=29 -y
    .... ....
    Complete!
    Download complete! Use 'dnf system-upgrade reboot' to start the upgrade.
    To remove cached metadata and transaction use 'dnf system-upgrade clean'
    The downloaded packages were saved in cache until the next successful transaction.
    You can remove cached packages by executing 'dnf clean packages'.

然后执行重启升级更新命令：

    # dnf system-upgrade reboot

饭毕 🍚 回来后就升级好了，最后清理升级前下载的安装包、无用的软件包：

    # dnf system-upgrade clean
    # dnf clean packages
    # dnf autoremove

参考：

[How to list, import and remove archive signing keys on CentOS 7 2016-01-21](https://linuxconfig.org/how-to-list-import-and-remove-archive-signing-keys-on-centos-7)

<br/>

本文标题 | [{{ page.title }}]({{ page.url }})
-------- |:--------
原始链接 | <{{ site.url }}{{ page.url }}>
