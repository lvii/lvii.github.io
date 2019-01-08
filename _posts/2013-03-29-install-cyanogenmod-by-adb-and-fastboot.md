---
layout: post
title: linux 下用 adb 和 fastboot 刷 CM10.1 M2
category: soft
tags: [android]
---

## fastboot 方式刷机

之前刷官方 rom tgz 包中的 `flash-all.sh` 脚本里面就这几句

    fastboot flash bootloader bootloader-maguro-primelc03.img
    fastboot reboot-bootloader
    sleep 5
    fastboot flash radio radio-maguro-i9250xxlf1.img
    fastboot reboot-bootloader
    sleep 5
    fastboot -w update image-yakju-jro03c.zip

fastboot 方式刷机，只要用 adb 连接设备，然后用 fastboot 开刷即可

把原生的 android 4.2.2 刷成 CyanogenMod 要使用 recovery **卡刷 zip 文件** 方式

## 绊脚石

### 找不到开发者选项

android 4.2.2 后 **开发者选项** 在 **设置** 中找不到了

参考这篇文章找到肾痛的打开方式：[Android 4.2 开发者选项在哪里 Developer options](http://blog.csdn.net/yajun0601/article/details/8622018)

    设置 --> 关于手机 --> 版本号

狂点 10 次打开 **开发者模式**  设置中出现 **开发者选项** 菜单后，进去勾选 **USB 调试**

![android enable develop mode](http://img.my.csdn.net/uploads/201302/28/1362033573_7558.png)

### 证书认证

开启 USB 调试后 adb devices 竟然提示 `offline`

    $ sudo ./adb devices
    * daemon not running. starting it now on port 5037 *
    * daemon started successfully *
    List of devices attached
    014E0FFD09010328        offline

[ADB No Longer Working on Android 4.2.2 ? - Update your ADB!](http://forum.xda-developers.com/showthread.php?t=2144709) 帖子提到：

> `4.2.2` now enforces **RSA authentication** via ADB

![android RSA auth](https://i.imgur.com/2fGhRXXl.jpg)

android 4.2.2 强制 **证书验证** 后才能使用 adb 连接设备，需要 **更新 adb 至新版** 才行

#### 安装 adb / fastboot ( NO NEED SDK )

刷机没有必要安装 **ADT Bundle SDK** 接近 400M 的大家伙， **SDK Tools Only** (87M) 中也没有 `adb` 和 `fastboot`

SDK Tools 中的 `tools/adb_has_moved.txt` 提到需要安装 `Android SDK Platform-tools` 才能获取 `adb` 和 `fastboot`

杯具的是 Platform-tools 选项是给 Eclipse 当插件勾选下载的，参考 archlinux AUR 仓库，找到实际下载链接：

    https://aur.archlinux.org/packages/android-sdk-platform-tools/
    https://dl-ssl.google.com/android/repository/platform-tools_r16-linux.zip

下载压缩包，解压后 adb 和 fastboot 一共也就才 2M 大小，杀鸡何必用牛刀  -_-#

    $ unzip -l platform-tools_r16-linux.zip|egrep -w 'adb|fastboot'
      1226659  11-10-2012 05:52   platform-tools/adb
       176294  11-10-2012 05:52   platform-tools/fastboot

    $ unzip platform-tools_r16-linux.zip  platform-tools/adb platform-tools/fastboot
    Archive:  platform-tools_r16-linux.zip
      inflating: platform-tools/adb
      inflating: platform-tools/fastboot

android 要把 user 整成 developer 呀，下个刷机工具都这么肾痛。 **注意** `adb` 和 `fastboot` 是 **32 位程序**

    $ file platform-tools/adb
    platform-tools/adb: ELF 32-bit LSB executable, Intel 80386, version 1 (SYSV)
    dynamically linked (uses shared libs), for GNU/Linux 2.6.8, not stripped

64bit 的 gentoo linux 需要安装 32bit 的 `emul-linux-x86-baselibs` lib 库才行

    $ sudo emerge -avj app-emulation/emul-linux-x86-baselibs

    $ ldd adb
        linux-gate.so.1 (0xf77d6000)
        librt.so.1 => /lib32/librt.so.1 (0xf77ae000)
        libncurses.so.5 => /lib32/libncurses.so.5 (0xf7769000)
        libpthread.so.0 => /lib32/libpthread.so.0 (0xf774f000)
        libstdc++.so.6 => /usr/lib/gcc/x86_64-pc-linux-gnu/4.6.3/32/libstdc++.so.6 (0xf7665000)
        libm.so.6 => /lib32/libm.so.6 (0xf763e000)
        libgcc_s.so.1 => /usr/lib/gcc/x86_64-pc-linux-gnu/4.6.3/32/libgcc_s.so.1 (0xf7622000)
        libc.so.6 => /lib32/libc.so.6 (0xf7498000)
        libdl.so.2 => /lib32/libdl.so.2 (0xf7494000)
        /lib/ld-linux.so.2 (0xf77d7000)

    $ ldd fastboot
        linux-gate.so.1 (0xf776e000)
        libstdc++.so.6 => /usr/lib/gcc/x86_64-pc-linux-gnu/4.6.3/32/libstdc++.so.6 (0xf7666000)
        libm.so.6 => /lib32/libm.so.6 (0xf763f000)
        libgcc_s.so.1 => /usr/lib/gcc/x86_64-pc-linux-gnu/4.6.3/32/libgcc_s.so.1 (0xf7623000)
        libc.so.6 => /lib32/libc.so.6 (0xf7498000)
        /lib/ld-linux.so.2 (0xf776f000)

搞定 `adb` 和 `fastboot` 后 `sudo ./adb kill-server` 关掉之前的 `adb server`

数据线连接好手机，使用最新安装的 adb 来确认证书

#### root 权限

遇到 `no permissions` 提示，需要使用 root 权限执行 `adb`

    · ./adb devices
    * daemon not running. starting it now on port 5037 *
    * daemon started successfully *
    List of devices attached
    ????????????    no permissions

#### 证书确认

使用 root 用户运行 adb 后，会提示 `unauthorized` 此时手机上面会弹出 **证书确认** 提示框

    gentoo platform-tools # ./adb devices
    * daemon not running. starting it now on port 5037 *
    * daemon started successfully *
    List of devices attached
    014E0FFD09010328        unauthorized

点击确认之后 adb 就可以正常识别到设备：

    gentoo platform-tools # ./adb devices
    List of devices attached
    014E0FFD09010328        device

## tips

连接设备后，可以用着两个命令快速进入 bootloader 和 recovery 而不用去按手机按键

    $ ./adb reboot recovery
    $ ./adb reboot bootloader

`sudo adb devices` 启动 `adb server` 时需要 root 权限，上面的 adb 操作普通权限即可

## 开刷

后面就可以按照 [CM 官方 WIKI](http://wiki.cyanogenmod.org/w/Install_CM_for_maguro) 按部就班的刷机了，主要有下面几步：

1. 解锁 bootloader
2. 刷 ClockworkMod Recovery
3. 复制 CM rom 和 gapps 到 sdcard
4. 进入 recovery 刷机

详细步骤还是参考 wiki 吧，刷机中遭遇下面几个问题：

1. `adb push` 复制 rom 文件时，路径后面需要有 `/` 不然会失败

```
$ ./adb push ../cm-10.1-20130304-EXPERIMENTAL-maguro-M2.zip /sdcard
failed to copy '../cm-10.1-20130304-EXPERIMENTAL-maguro-M2.zip' to '/sdcard': Is a directory

$ ./adb push ../cm-10.1-20130304-EXPERIMENTAL-maguro-M2.zip /sdcard/
5113 KB/s (168244318 bytes in 32.133s)
```

2. 默认 CyanogenMod 没有 google play store 没法下载应用，需要安装 [gapps](http://goo.im/gapps) ：

> Optional:  
> Place any supplemental packages (eg Google Apps or kernel)
> .zip file(s) on the root of the SD card.

刷完后只有 google 服务框架 / google play / gtalk / 搜索 / 邮件 / 天气 几个应用

gmail / map / plus / youtube / now / ... 什么的都没有，很干净




<br/>

本文标题 | [{{ page.title }}]({{ page.url }})
-------- |:--------
原始链接 | <{{ site.url }}{{ page.url }}>
