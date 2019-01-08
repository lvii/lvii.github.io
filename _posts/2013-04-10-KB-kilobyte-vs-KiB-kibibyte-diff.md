---
layout: post
title: KB kilobyte 和 KiB kilibyte 单位的区别
category: system
tags: [unit, standard]
---

## kickass

[A description of the coreutils numfmt utility](http://www.pixelbeat.org/docs/numfmt.html) 文中介绍了 `numfmt` 工具

`numfmt` 主要用来 **格式化数字** 为 **human readable** 格式

`numfmt` 的 `--to=si` 和 `--to=iec-i` 参数指定了两种不同的 **单位标准** 转换数字

就以 **si** 和 **iec** 为关键词开始扫盲，搞清楚 `1024` 和 `1000` 两种进制的不同

## 标准 standard

[GUN numfmt 手册](http://www.gnu.org/software/coreutils/manual/html_node/numfmt-invocation.html) 中提到 unit 单位的 3 个标准：`si / iec / iec-i`

    si
        International System of Units (SI) standard 'K'  =>  1000^1 = 10^3 (Kilo)
    iec
        International Electronical Commission (IEC) standard 'K'  =>  1024^1 = 2^10 (Kibi)
        The iec option uses a single letter suffix (e.g. 'G')
        which is not fully standard as the iec standard recommends a
        two-letter symbol (e.g 'Gi') - but in practice, this method common.
    iec-i
        International Electronical Commission (IEC) standard 'Ki'  =>  1024^1 = 2^10 (Kibi)
        The iec-i option uses a two-letter suffix symbol (e.g. 'Gi'),
        as the iec standard recommends, but this is not always common in practice.

看了上文知道了 2 个不同的单位标准：**SI 国际单位制** 和 **IEC 国际电工二进制**

## 单位 unit

但还需要再脑补一下 `KB` 和 `KiB` 这俩有啥不同，继续百科它们：

[数据数量级 Orders of magnitude (data)](https://en.wikipedia.org/wiki/Orders_of_magnitude_(data))

> 1 kB (kilobyte) = 1000 bytes = 8000 bits  
> 1 KiB (kibibyte) = 2^10 bytes = 1024 bytes = 8192 bits

[二进制乘数词头](https://zh.wikipedia.org/wiki/二进制乘数词头)

> 1999 年 1 月，国际电工委员会（IEC）引入了 "kibi-"、"mebi-"、"gibi-" 等词头  
> 以及缩写符号 "Ki"、"Mi"、"Gi" 等来明确说明二进制乘数计数。名字的前两个字母  
> 来源于原来的国际单位制词头，而后面的 "bi" 是二进制的缩写

    | decimal 1000 | binary 1024 |
    |--------------+-------------|
    | kilobyte     | kibibyte    |
    | megabyte     | mebibyte    |
    | gigabyte     | gibibyte    |
    | terabyte     | tebibyte    |
    | petabyte     | pebibyte    |
    | exabyte      | exbibyte    |
    | zettabyte    | zebibyte    |
    | yottabyte    | yobibyte    |

这样看来 KiB / MiB / GiB 才是标准的 2 进制单位 KB / MB / GB 是 10 进制的单位

虽然这个标准出了又快 15 年了，但是 linux 下面有些工具并木有都按照标准来

## 工具 tool

平时常用的磁盘相关的工具 `fdisk` / `parted` / `lvs` / `lvcreate` 的 man 手册：

`fdisk` 默认用的是 10 进制，如果转换成 GiB 需要自己手动用 bytes 值计算

    -u, --unit letter
        Interpret the input and show the output in the units specified by letter.
        This letter can be one of S, C, B or M, meaning Sectors,
        Cylinders, Blocks and Megabytes, respec‐tively.
        The default is cylinders, at least when the geometry is known.

`parted` 已 **标准化**

    unit unit
        Set unit as the unit to use when displaying locations and sizes,
        and for inter‐preting those given by the user when not suffixed
        with an explicit unit. unit can be one of "s" (sectors), "B" (bytes),
        "kB", "MB", "MiB", "GB", "GiB", "TB", "TiB", "%" (percentage
        of device size), "cyl" (cylinders), "chs" (cylinders, heads, sec‐tors),
        or "compact" (megabytes for input, and a human-friendly form for output).

`lvs` 的 `--unit` 选项，大写字母为 10 进制，小写字母为 2 进制单位

    --units hHbBsSkKmMgGtTpPeE
        All sizes are output in these units:
        (h)uman-readable, (b)ytes, (s)ectors, (k)ilobytes, (m)egabytes,
        (g)igabytes, (t)erabytes, (p)etabytes, (e)xabytes.
        Capitalise to use multiples of 1000 (S.I.) instead of 1024.

`lvcreate` 这货，直接无视上面自家兄弟的大小写规则，全部都是以 2 进制来搞的

    -L, --size LogicalVolumeSize[bBsSkKmMgGtTpPeE]
        Gives the size to allocate for the new logical volume.
        A size suffix of K for kilobytes, M  for megabytes, G for gigabytes,
        T for terabytes, P for petabytes or E for exabytes is optional.
        Default unit is megabytes.

其中只有 `parted` 对标准支持比较好，其他的就比较杯具了

再黑一下 LVM 查看逻辑卷可以使用 10 进制，但是 **创建** 的时候就没有 10 进制单位选项了 -_-#

还有像 `ifconfig` 等查看 **网络流量** 的 linux 工具，也需要注意下单位标准 ...






<br/>

本文标题 | [{{ page.title }}]({{ page.url }})
-------- |:--------
原始链接 | <{{ site.url }}{{ page.url }}>
