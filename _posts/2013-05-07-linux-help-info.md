---
layout: post
title: linux 帮助的那些事
category: system
tags: [help, man, info]
---

熟悉使用一个工具的大概阶段：

- 知道这货是做甚的？
- **just use it**
- 不断重复的：**发现**、**使用** 它的新功能

在深入使用的过程中，会遭遇各种莫名的参数，还想知道工具是否具有自己期望的功能，
可以从系统的帮助获取一些有用的信息。之前入门 linux 看过一些书籍，很少有介绍如何获取 linux 帮助的，
大多仅仅介绍 `man` 和 `info` 两个命令基础使用，有种 **授人以鱼** 的感觉。
有些 Linux 的 app 其实是没有提供 `man` 或 `info` 手册，之前折腾的 irssi xmpp 插件就是这种情况。

粗略整理了下我知道的几类 linux 的帮助信息 (根据使用频率排序)：

- `command --help` 或 `-h`
- `man`
- `help bash_buildin`
- `/usr/share/doc/` 目录下囧异的各色文件格式
- `info`

思维导图如下图所示：

![linux-help-info-mindmap](http://fc02.deviantart.net/fs71/f/2013/127/a/5/linux_help_info_mindmap_by_57lvii-d64exob.png)

linux 下像 `time` 这样 `info` / `man` / `doc` 文档一应俱全的 **有业界良心** 的程序不多 ...

    $ dpkg -L time
    /.
    /usr
    /usr/bin
    /usr/bin/time
    /usr/share
    /usr/share/info
    /usr/share/info/time.info.gz
    /usr/share/doc
    /usr/share/doc/time
    /usr/share/doc/time/AUTHORS
    /usr/share/doc/time/README
    /usr/share/doc/time/time.html
    /usr/share/doc/time/copyright
    /usr/share/doc/time/NEWS.gz
    /usr/share/doc/time/changelog.Debian.gz
    /usr/share/doc/time/changelog.gz
    /usr/share/doc-base
    /usr/share/doc-base/time
    /usr/share/man
    /usr/share/man/man1
    /usr/share/man/man1/time.1.gz

## `cmd --help | -h`

找 help 的第一条件反射就是敲 `--help | -h`，可惜不是所有 app 都会注释每个参数的作用

    $ ssh --help
    usage: ssh [-1246AaCfgKkMNnqsTtVvXxYy] [-b bind_address] [-c cipher_spec]
               ... ...
               [user@]hostname [command]

`[-1246AaCfgKkMNnqsTtVvXxYy]` 不传值的参数没 help 说明，想知道参数作用只能查 man 手册了

## `man`

其实 man 不是男人的意思，是 **manual** 的缩写，，，囧rz，下面是使用 man 的 2 个技巧吧

### 查询 | 搜索

有两个查找 man 的命令：`apropos` 和 `whatis`:

| command | equivalent       | description                                    |
|-------- | ---------------- | -----------------------------------------------|
| `apropos` | `man -k <keyword>` | **模糊匹配** 关键词 (search the whatis database for **strings**)
| `whatis`  | `man -f <keyword>` | **精确匹配** 关键词 (search the whatis database for **complete words**)

```
$ apropos vim
rvim (1)             - Vi IMproved, a programmers text editor
vim (1)              - Vi IMproved, a programmers text editor
vimdiff (1)          - edit two, three or four versions of a file with Vim and show differences
vimtutor (1)         - the Vim tutor

$ whatis vim
vim (1)              - Vi IMproved, a programmers text editor
```

`man whatis` 有提到 `whatis database` 数据库：

>   the `whatis` database is created using the command `/usr/sbin/makewhatis`

`whatis` 和 `apropos` 都查询这个数据库 `makewhatis` 创建之，但不知如何判断数据库是否存在？

### 章节

man 手册还会分几个章节介绍命令：

| SECTIONS | Distributions                                  |
|--------- | -----------------------------------------------|
|        1 | User Commands                                  |
|        2 | System Calls                                   |
|        3 | C Library Functions                            |
|        4 | Devices and Special Files                      |
|        5 | File Formats and Conventions                   |
|        6 | Games et. Al.                                  |
|        7 | Miscellanea `man(7)`                           |
|        8 | System Administration tools and Deamons        |

`whatis time` 可以搜到下面几个不同的章节:

    $ whatis time
    time                 (1)  - time a simple command or give resource usage
    time                 (2)  - get time in seconds
    time                 (7)  - overview of time and timers

默认 `man time` 查看的手册的左上角 `TIME(1)` 表示当前所在的是第 1 章

可用 `man 7 time` 查看第 7 章的帮助手册

## `help builtin`

如果使用 bash shell 对于 bash 的内建命令可以使用 `help buildin_command` 来获取帮助信息
在写脚本时，忘了 `if` 判断的某个参数代表的意思，可以 `help test` 一下，就可以
输出 `if` 比较 **文件 / 字符串 / 整数** 的各个参数的帮助信息。
判断一个命令是否是 bash 关键字或内建命令，可以使用 `type -a <command>` 命令：

    $ type -a test
    test is a shell builtin
    test is /usr/bin/test

    $ type -a time
    time is a shell keyword
    time is /usr/bin/time

    $ type -a kill
    kill is a shell builtin
    kill is /bin/kill
    kill is /usr/bin/kill

默认 bash 调用上面几个命令，优先使用内建关键字，如果想使用外部程序，加上 **绝对路径** `/usr/bin/time` 即可

## `/usr/share/doc/`

终于到这个不常用的目录了 gentoo 安装程序时，有个 `doc` USE，像 python
这种有官方文档的应用，如果启用这个 USE 会把文档安装在这里吧。
之前折腾的 irssi xmpp 插件木有 help 木有 man 但是有 document :

    $ equery f irssi-xmpp
    ... ...
    /usr/share/doc/irssi-xmpp-0.52
    /usr/share/doc/irssi-xmpp-0.52/FAQ.bz2
    /usr/share/doc/irssi-xmpp-0.52/GENERAL.bz2
    /usr/share/doc/irssi-xmpp-0.52/INTERNAL.bz2
    /usr/share/doc/irssi-xmpp-0.52/MUC.bz2
    /usr/share/doc/irssi-xmpp-0.52/NEWS.bz2
    /usr/share/doc/irssi-xmpp-0.52/README.bz2
    /usr/share/doc/irssi-xmpp-0.52/STARTUP.bz2
    /usr/share/doc/irssi-xmpp-0.52/TODO.bz2
    /usr/share/doc/irssi-xmpp-0.52/XEP.bz2

要直接查看 doc 还是有点障碍的 document 会有多种不同的文件格式 `txt/html/pdf/bz2/gz/xz/gs/tex...`

参考这篇文章：[zutils: zcat and friends on Steroids](http://noone.org/blog/English/Computer/Debian/CoolTools)
查看压缩过的文档，需要用到 `bzcat` 或 `zless` 命令

| default | `gzip`   | `bzip2`   | `lzma`    | `xz`      |
|-------- | -------- | --------- | --------- | ----------|
| `cat`   | `zcat`   | `bzcat`   | `lzcat`   | `xzcat`   |
| `cmp`   | `zcmp`   | `bzcmp`   | `lzcmp`   | `xzcmp`   |
| `diff`  | `zdiff`  | `bzdiff`  | `lzdiff`  | `xzdiff`  |
| `grep`  | `zgrep`  | `bzgrep`  | `lzgrep`  | `xzgrep`  |
| `egrep` | `zegrep` | `bzegrep` | `lzegrep` | `xzegrep` |
| `fgrep` | `zfgrep` | `bzfgrep` | `lzfgrep` | `xzfgrep` |
| `more`  | `zmore`  | `bzmore`  | `lzmore`  | `xzmore`  |
| `less`  | `zless`  | `bzless`  | `lzless`  | `xzless`  |

查看 irssi xmpp 插件的文档：

    $ bzcat /usr/share/doc/irssi-xmpp-0.52/STARTUP.bz2|less

## info

info 比较适合 emacs 粉，不过它的一些导航命令好像混合了 vi 和 emacs 也是一朵奇葩，听说有些命令的细则，可以从这里面找到。没怎么用过 info 所以等用到时再做补充，，，

## 尾声

简单的整理，希望能 **授人以渔**，对 linux 命令的使用，帮助手册有些仅仅介绍参数的含义，
有些会附带一些示例。快速入门的话最好 google 一些 tutorial 或 how to 快速了解该命令的常见用法，
然后对个别有疑问的参数，就可以从上面几个地方获取帮助信息



<br/>

本文标题 | [{{ page.title }}]({{ page.url }})
-------- |:--------
原始链接 | <{{ site.url }}{{ page.url }}>
