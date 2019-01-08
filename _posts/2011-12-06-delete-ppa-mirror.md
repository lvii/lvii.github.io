---
layout: post
title: ppa 源删除
category: system
tags: [ppa, mirror, ubuntu]
---

ubuntu 的 ppa 源都被放在 `/etc/apt/sources.list.d` 目录

升级到 oneiric 版本后，之前添加的是 natty 的源，升级后 ppa 源没有自动升级

只能手动删除，重新添加一遍或手工修改

[Removing apt repository in ubuntu karmic koala](http://www.kelpdesign.com/tech-talk/remove-apt-repository-in-karmic)
这篇文章的评论里中提到:

ubuntu 10.10 后 `add-apt-repository` 有个 **删除选项** `-r` 参数

`man` 了下 `add-apt-repository` 没找到删除参数，原来在 `--help` 里面：

    -r, --remove    remove repository from sources.list.d directory

    # sudo add-apt-repository -r ppa:ferramroberto/java
    Error: 'deb http://ppa.launchpad.net/ferramroberto/java/ubuntu oneiric main' doesn't exist in a sourcelist file

看来 `add-apt-repository` 不是很完善，只能 **手动删除** `/etc/apt/sources.list.d/` 下的文件了！












<br/>

本文标题 | [{{ page.title }}]({{ page.url }})
-------- |:--------
原始链接 | <{{ site.url }}{{ page.url }}>
