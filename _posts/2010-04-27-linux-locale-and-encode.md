---
layout: post
title: linux 文件编码 字符集 语言标志 编码转换
category: code
tags: [locale, utf8, unicode, i18n]
---

> 中文就是永远的痛！ —— 《可爱的python》

中文乱码有时让人纠结啊，下面就说说 linux 下文件编码，字符集，语言标志，编码转换种种

## locale

第一个相关的应该是 locale 相关的环境变量吧：

[wikipedia Locale](http://en.wikipedia.org/wiki/Locale)

> On POSIX platforms such as Unix, Linux and others, locale identifiers are defined similar
> to the BCP 47 definition of language tags, but the locale variant modifier is defined differently,
> and the character set is included as a part of the identifier.
> It is defined in this format: `[language[_territory][.codeset][@modifier]]`. (For example, Australian English using the UTF-8 encoding is `en_AU.UTF-8`.)

locale 命名规则 `[语言]_[地区].[字符集编码]` 如 `en_US.UTF-8 , zh_CN.UTF-8`

locale 设置有问题，终端显示输出会出现乱码，继承 locale 环境变量的程序也会有问题

## 文件编码

- 编辑器 (vim, emacs) 保存文本编码，浏览器 (firefox) 网页编码
- 数据库 (mysql) 对数据的存储，传递，显示相关编码
- 编程语言 (python, ruby) 对中文编码 ... 编码

基本只要碰到中文，就要难免会遭遇乱码，特别是和 windows 之间有文本交互时

windows 下编码看起来不是主流的 GBK 或 GB2312 格式，而是莫名奇妙的 `CP936` **代码页**

[wikipedia 区域设置](http://zh.wikipedia.org/wiki/区域设置)

> Windows 操作系统的简体中文的默认 **编码字符集**（即 **代码页**）是 `GBK` ，即 **"936 (ANSI/OEM - Simplified Chinese GBK)"**

在 vim 中查看 GBK 编码的文件得到的也是这种莫名的格式：

    :set fileencoding?
    fileencoding=cp936

字符集 | 关于 | 代码页 (CodePage) | 关于
------ | ---- | ----------------- | ----
`UTF-8` | 一个 unicode 编码子集 | `CP65001` | 对应 UTF-8 Unicode
`GBK` | `GB2312` 字符集的扩展 | `CP936` | 简体中文 Simplified Chinese GBK
`GB2312` | 简体中文下 ANSI 编码标准 | `CP950` | 繁体中文 Traditional Chinese Big5

详细见 [微软 MSDN 官方介绍](http://msdn.microsoft.com/en-us/goglobal/bb964654.aspx)

windows cmd 下查看 MySQL 表数据乱码，可以用 `chcp 936` 配合 `set names gbk ` 修复乱码

## 编码转换

linux 下的 `iconv` 可以修改文件编码，可打印输出，也可以导出到另外一个文件

    iconv -f encoding -t encoding inputfile
    iconv -f GBK -t UTF-8 inputfile -o  outputfile

GBK/GB2312 结果都一样，都可以显示正常

`enca` 工具也可以实现文件编码分析判断以及编码转换，可能需要另行安装：

列出所支持的语言和相应字符集：

    $ enca --list languages
        ... ...
        russian: KOI8-R CP1251 ISO-8859-5 IBM866 maccyr
        chinese: GBK BIG5 HZ
           none:

判断文件编码 `enca -L zh filename` ：

    $ file README.rst
    README.rst: UTF-8 Unicode text

    $ enca -L zh README.rst
    Universal transformation format 8 bits; UTF-8

    $ file ANSI.txt
    ANSI.txt: ISO-8859 text, with CRLF line terminators

    $ enca -L zh ANSI.txt
    Simplified Chinese National Standard; GB2312
    CRLF line terminators

将文件转换为 UTF-8 编码 `enca -L zh_CN -x UTF-8 filename` ：

    $ enca -L zh -x UTF-8 ANSI.txt
    $ file ANSI.txt
    ANSI.txt: UTF-8 Unicode text

<br/>

本文标题 | [{{ page.title }}]({{ page.url }})
-------- |:--------
原始链接 | <{{ site.url }}{{ page.url }}>
