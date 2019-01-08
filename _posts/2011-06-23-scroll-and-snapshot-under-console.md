---
layout: post
title: console 控制台下翻页，截屏
category: system
tags: [console, tips]
---

之前有遇到 kernel panic 和 lvm 自检错误，导致无法进入 X 。
更杯具的是错误信息滚屏了，鼠标，翻页键都没法往上滚屏。

请教前辈得知 console 控制台下的翻页快捷键：

    Shift + PageUp / PageDn

如果 X down 了，在 console 下排错后 需要对一些输出信息进行保存，但是没法用截屏工具。

推荐 `setterm` 工具的 `-dump` 参数，man 手册描述：

    -dump [1-NR_CONS]
        Writes a snapshot of the given virtual console (with attributes) to the file
        specified in the -file option, overwriting its contents; the default is screen.dump.
        Without an argument, dumps the current virtual console. Overrides -append.

在当前 console 下执行 `setterm -dump` 会保存当前屏幕的输出到 `screen.dump` 文件

如果要保存的输出在上面需要滚动到相应位置，但是不能再在当前 console 输入命令保存截屏了

需要 Ctrl-Alt-F1~F6 切换到其他 console 下登录执行：

    setterm -dump <console_NUM>

来保存对应 `<console_NUM>` 的屏幕输出到 `screen.dump` 文件

<br/>

本文标题 | [{{ page.title }}]({{ page.url }})
-------- |:--------
原始链接 | <{{ site.url }}{{ page.url }}>
