---
layout: post
title: "CentOS 8 都更新了些什么"
category: system
tags: [centos]
---

# WHEN

<https://wiki.centos.org/About/Building_8>

`2019-09-24` 基于 RHEL8 的 CentOS 8 发布了

RHEL 8.1 beta 也出了一段时间，这次大版本的更新变化的地方还是蛮多的

# WHAT

## mirrorlist

`AppStream` 源替换了原来的 `updates` 源，还多了 module 相关的模块化安装：

    # cd /etc/yum.repos.d/ && grep ^mirrorlist CentOS-*
    CentOS-AppStream.repo:mirrorlist=http://mirrorlist.centos.org/?release=$releasever&arch=$basearch&repo=AppStream&infra=$infra
    CentOS-Base.repo:mirrorlist=http://mirrorlist.centos.org/?release=$releasever&arch=$basearch&repo=BaseOS&infra=$infra
    CentOS-centosplus.repo:mirrorlist=http://mirrorlist.centos.org/?release=$releasever&arch=$basearch&repo=centosplus&infra=$infra
    CentOS-Extras.repo:mirrorlist=http://mirrorlist.centos.org/?release=$releasever&arch=$basearch&repo=extras&infra=$infra
    CentOS-fasttrack.repo:mirrorlist=http://mirrorlist.centos.org/?release=$releasever&arch=$basearch&repo=fasttrack&infra=$infra
    CentOS-PowerTools.repo:mirrorlist=http://mirrorlist.centos.org/?release=$releasever&arch=$basearch&repo=PowerTools&infra=$infra

    http://mirrorlist.centos.org/?release=8&arch=x86_64&repo=AppStream
    http://mirrorlist.centos.org/?release=8&arch=x86_64&repo=BaseOS
    http://mirrorlist.centos.org/?release=8&arch=x86_64&repo=centosplus
    http://mirrorlist.centos.org/?release=8&arch=x86_64&repo=extras
    http://mirrorlist.centos.org/?release=8&arch=x86_64&repo=fasttrack
    http://mirrorlist.centos.org/?release=8&arch=x86_64&repo=PowerTools

<https://developers.redhat.com/blog/2018/11/15/rhel8-introducing-appstreams/>

## yum module

<https://docs.centos.org/en-US/8-docs/managing-userspace-components/assembly_finding-rhel-8-content/>

直接用 `yum search` 是找不到 `ipa-server` 的，被放入到了 `idm:DL1/server` module ：

    # yum module list idm:DL1/server
    CentOS-8 - AppStream
    Name        Stream        Profiles                                       Summary
    idm         DL1           common [d], adtrust, client, dns, server       The Red Hat Enterprise Linux Identity Management system module

    Hint: [d]efault, [e]nabled, [x]disabled, [i]nstalled

    # yum module info idm:DL1/server|head
    Last metadata expiration check: 1:02:17 ago on Mon 14 Oct 2019 03:58:38 PM CST.
    Ignoring unnecessary profile: 'idm/server'
    Name             : idm
    Stream           : DL1
    Version          : 8000020190628154621
    Context          : e0ee5dbd
    Profiles         : common [d], adtrust, client, dns, server
    Default profiles : common
    Repo             : AppStream
    Summary          : The Red Hat Enterprise Linux Identity Management system module

    # yum module info --profile idm:DL1/server
    Last metadata expiration check: 0:24:39 ago on Mon 14 Oct 2019 03:58:38 PM CST.
    Ignoring unnecessary profile: 'idm/server'
    Name    : idm:DL1:8000020190628154621:e0ee5dbd:x86_64
    common  : ipa-client
    adtrust : ipa-idoverride-memberof-plugin
            : ipa-server-trust-ad
    client  : ipa-client
    dns     : ipa-server
            : ipa-server-dns
    server  : ipa-server                        <-- ipa-server

## rpm SPEC

<https://git.centos.org/rpms/nginx/branches>

## grub

内核引导的配置从 `/boot/grub2/grub.cfg` 文件分离出来，放到 `/boot/loader/entries/` 目录：

    # tree -F /boot/loader/
    /boot/loader/
    └── entries/
        ├── bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb-0-rescue.conf
        ├── bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb-4.18.0-80.el8.x86_64.conf
        └── e9218483634816ff3282777cd960dedf-5.2.0-rc3.conf

    1 directory, 3 files

    # cat /boot/loader/entries/e9218483634816ff3282777cd960dedf-5.2.0-rc3.conf
    title CentOS Linux (5.2.0-rc3) 8 (Core)
    version 5.2.0-rc3
    linux /vmlinuz-5.2.0-rc3
    initrd /initramfs-5.2.0-rc3.img $tuned_initrd
    options $kernelopts $tuned_params
    id centos-20191007003758-5.2.0-rc3
    grub_users $grub_users
    grub_arg --unrestricted
    grub_class kernel

默认引导内核：

    # grubby --default-index
    0

    # grubby --default-kernel
    /boot/vmlinuz-5.2.0-rc3

    # grubby --info=ALL|awk -F\" '$1=="kernel=" {print i++ " : " $2}'
    0 : /boot/vmlinuz-5.2.0-rc3
    1 : /boot/vmlinuz-4.18.0-80.el8.x86_64
    2 : /boot/vmlinuz-0-rescue-bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb

[monitoring and updating the kernel Chapter 3. Configuring kernel command line parameters](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/managing_monitoring_and_updating_the_kernel/configuring-kernel-command-line-parameters_managing-monitoring-and-updating-the-kernel#what-are-boot-entries_configuring-kernel-command-line-parameters)

修改（增/删）内核引导参数，可以用 `grub2-editenv` 命令：

    grub2-editenv - set "$(grub2-editenv - list | grep kernelopts) <NEW_PARAMETER>"

    grub2-editenv - set "$(grub2-editenv - list | grep kernelopts | sed -e 's/<PARAMETER_TO_REMOVE>//')"

也可以使用 `grubby` 命令，可以参考：

[26.4. Making Persistent Changes to a GRUB 2 Menu Using the grubby Tool](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/system_administrators_guide/sec-making_persistent_changes_to_a_grub_2_menu_using_the_grubby_tool)

    grubby --remove-args="argX argY" --args="argA argB" --update-kernel /boot/kernel

## kernel

内核的一些新特性：

- CGroups V2
- Overlay FS2
- TCP BBR
- ePBF / XDP

### ePBF / XDP

[Cilium 1.6: KVstore-free operation, 100% kube-proxy replacement, Socket-based load-balancing, ... 2019-08-20](https://cilium.io/blog/2019/08/20/cilium-16/)

[Kubernetes NodePort (beta)](https://docs.cilium.io/en/latest/gettingstarted/nodeport/)

> Cilium to enable Kubernetes **NodePort** services in BPF which can replace NodePort implemented by `kube-proxy`.
> Enabling the feature allows to run a fully functioning Kubernetes cluster **without** `kube-proxy`.
> NodePort services depend on the [Host-Reachable Services (beta)](https://docs.cilium.io/en/latest/gettingstarted/host-services/#host-services) feature, therefore a **v4.19.57**, v5.1.16, v5.2.0 or more recent Linux kernel is required.

CentOS 8 系的内核版本 `4.18.0` 有点旧，不知未来 redhat 是否会 backport 新内核特性：

    # rpm -q --queryformat="%{VERSION}\n" kernel
    4.18.0

### CGroups V2

[World domination with cgroups in RHEL 8: welcome cgroups v2! ](https://www.redhat.com/en/blog/world-domination-cgroups-rhel-8-welcome-cgroups-v2)

[Migrating from CGroups V1 in Red Hat Enterprise Linux 7 and below to CGroups V2 in Red Hat Enterprise Linux 8](https://access.redhat.com/articles/3735611)

![img](https://access.redhat.com/sites/default/files/images/cgroupsv1.png)

> Above, the `bg` cgroup is associated with the `blkio` and memory cgroups in V1 but is a **duplicated** entry in the hierarchy since the `bg` cgroup is associated with multiple controllers.
> In V2, the `bg` cgroup exists once and has the `memory` and `io` controllers associated with it as described by `cgroup.subtree_control`.

内核引导增加 `systemd.unified_cgroup_hierarchy=1` 参数即可启用 CGroups V2 ：

    grubby --update-kernel=ALL --args="systemd.unified_cgroup_hierarchy=1"

    systemctl set-property user.slice CPUQuota=50%

## podman (docker/moby)

Redhat 造的替换 docker 的轮子 `podman` ：

- CGroups V2
- **非特权** 普通用户

[RHEL8 Building, running, and managing containers](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/building_running_and_managing_containers/)

[Containers without daemons: Podman and Buildah available in RHEL 7.6 and RHEL 8](https://developers.redhat.com/blog/2018/11/20/buildah-podman-containers-without-daemons/)

[Fedora 31: Docker package no longer available and will not run by default (due to switch to cgroups v2)](https://fedoraproject.org/wiki/Common_F31_bugs#docker-moby-engine)

> The Docker package has been **removed** from Fedora 31. It has been replaced by the upstream package `moby-engine`, which includes the **Docker CLI** as well as the **Docker Engine**. However, we recommend instead that you use `podman`, which is a **Cgroups v2-compatible** container engine whose CLI is compatible with Docker's. **Fedora 31 uses Cgroups v2 by default**. The `moby-engine` package does **NOT** support Cgroups v2 yet, so if you need to run the `moby-engine` or run the **Docker CE** package, then you need to switch the system to using Cgroups v1, by passing the kernel parameter `systemd.unified_cgroup_hierarchy=0`

<https://success.docker.com/article/compatibility-matrix>

Docker EE 支持 CentOS 8

## python

[Chapter 6. Using Python in Red Hat Enterprise Linux 8](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/configuring_basic_system_settings/using-python3_configuring-basic-system-setting)

`yum` 和 `dnf` 等系统工具对 python 的依赖跟 python 软件包解绑，搞了个 `platform-python` 马甲：

    # rpm -ql platform-python
    /usr/bin/pydoc3.6
    /usr/bin/pyvenv-3.6
    /usr/bin/unversioned-python
    /usr/lib/.build-id
    /usr/lib/.build-id/54
    /usr/lib/.build-id/54/58bd10988cc975783b2801fff1f2fa253375e2
    /usr/lib/.build-id/54/58bd10988cc975783b2801fff1f2fa253375e2.1
    /usr/libexec/no-python
    /usr/libexec/platform-python
    /usr/libexec/platform-python3.6
    /usr/libexec/platform-python3.6m
    /usr/share/doc/platform-python
    /usr/share/doc/platform-python/README.rst
    /usr/share/licenses/platform-python
    /usr/share/licenses/platform-python/LICENSE
    /usr/share/man/man1/python.1.gz
    /usr/share/man/man1/python3.6.1.gz
    /usr/share/man/man1/unversioned-python.1.gz

    # /usr/libexec/platform-python
    Python 3.6.8 (default, Oct  7 2019, 17:58:22)
    [GCC 8.2.1 20180905 (Red Hat 8.2.1-3)] on linux
    Type "help", "copyright", "credits" or "license" for more information.
    >>>

要用 python3 还是需要单独安装 `python36` 软件包：

    # rpm -q platform-python python36
    platform-python-3.6.8-4.el8_0.x86_64
    python36-3.6.8-2.module_el8.0.0+33+0a10c0e1.x86_64

<br/>

本文标题 | [{{ page.title }}]({{ page.url }})
-------- |:--------
原始链接 | <{{ site.url }}{{ page.url }}>
