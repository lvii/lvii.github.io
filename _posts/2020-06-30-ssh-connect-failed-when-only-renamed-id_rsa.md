---
layout: post
title: "私钥重命名为默认 id_rsa 导致 ssh 无法连接"
category: system
tags: [ssh]
---

# WHAT

连接多个机器的不同用户，使用的 `id_rsa` 私钥也不同，尝试将某一个 `id_rsa.user` 重命名为 **默认** 的 `id_rsa`，发现修改后 ssh 无法连接。

# WHY

查了一下发现同时存在 `id_rsa` 和 `id_rsa.pub` 两者必须要 **配对** 才能正常使用。

[sshd wont accept key but only if named id_rsa](https://superuser.com/questions/1424281/sshd-wont-accept-key-but-only-if-named-id-rsa)

[SSH connection with public key not working with specific name](https://unix.stackexchange.com/questions/557052/ssh-connection-with-public-key-not-working-with-specific-name)

# HOW

将 `id_rsa.pub` 重命名，只保留 `id_rsa` 就可以了。

<br/>

本文标题 | [{{ page.title }}]({{ page.url }})
-------- |:--------
原始链接 | <{{ site.url }}{{ page.url }}>