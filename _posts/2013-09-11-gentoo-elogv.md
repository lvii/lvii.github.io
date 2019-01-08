---
layout: post
title: gentoo 下使用 elogv 收集 emerge 安装软件的日志信息
category: system
tags: [gentoo]
---

## 起因

gentoo 升级 world 完后，有些软件包会输出一些更新操作信息。
有些操作被忽略了，后面运行出现问题，升级后没有保存软件包的提醒信息，没法查询就杯具了。
`elogv` 工具是专门用来收集 emerge 安装软件包的提示信息，定义不同的日志级别，可以记录不同的日志信息

## 经过

安装好 `elogv` 后，在 `/etc/make.conf` 中添加要记录的日志级别:

    PORTAGE_ELOG_SYSTEM="save"
    PORTAGE_ELOG_CLASSES="warn error log"

日志级别有下面几个级别：

- `error`
- `warn`
- `info`
- `log`
- `qa`

可以根据需要自行配置，仅仅用来收集软件包安装后的提示信息的话，使用 `warn` 和 `log` 即可

## 结果

百文不如一见，下面对比一下使用 `elogv` 收集日志前后的不同

没有启用 `elogv` 之前，安装 `net-misc/openssh` 输出下面提示：

![gentoo emerge openssh message][1]

[1]: http://media-cache-ec0.pinimg.com/736x/1f/2a/e8/1f2ae82b94446e5a29a2013f118010db.jpg

图中上面的 **绿色星号** 表示 log 信息，下面的 **黄色星号** 标注的是 `warning` 信息。

下面看看 `elogv` 中对应日志信息：

![gentoo elogv message][2]

[2]: http://media-cache-ec0.pinimg.com/originals/e0/80/d9/e080d90534ed55318d527d6d170388a9.jpg

类别 | 字段
---- | ----
**日志** | `LOG: postinst`
**警告** | `WARN: postinst`

查看较长的日志信息，使用 **空格** `<space>` 快捷键翻页



<br/>

本文标题 | [{{ page.title }}]({{ page.url }})
-------- |:--------
原始链接 | <{{ site.url }}{{ page.url }}>
