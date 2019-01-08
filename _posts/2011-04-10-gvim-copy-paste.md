---
layout: post
title: vim 复制/粘贴 代码格式混乱问题
category: soft
tags: [vim]
---

我比较常用 xterm 终端，每次在 vim 之间复制代码段

为了不使粘贴的文本格式乱掉，保留代码的原始格式，主要是缩进

要先关闭 vim 的 paste 选项，然后再用 `Shift + 鼠标选区` 选中要复制的文本

到要粘贴的文本处，使用 `Shift + Insert 或 鼠标中键` 进行粘贴

不然会出现缩进格式乱掉，虽然定义了 **快捷键** <F10> 但每次来回切换也不是很方便

    nnoremap <F10> :setlocal nu! list! paste!<CR>

为了更方便些，只好安装 gvim，使用 `gvim -v` 替换原来的 vim

之后 xterm 中的 vim 代码粘贴，不用再按住 Shift 进行鼠标选择

复制 / 粘贴 格式也没有问题了！




<br/>

本文标题 | [{{ page.title }}]({{ page.url }})
-------- |:--------
原始链接 | <{{ site.url }}{{ page.url }}>
