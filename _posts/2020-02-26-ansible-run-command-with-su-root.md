---
layout: post
title: "ansible 使用 su 切换到 root 执行命令"
category: system
tags: [ansible]
---

# WHAT

厂里一批机器 `/etc/sudoers.d/` 规则配置错误，导致无法使用 `sudo`

只能使用 `su - root` 切换到 `root` 移除错误的配置


# HOW

通过 `--become --become-user=root --become-method=su --ask-become-pass` 参数切换用户：

    ansible all -m raw -a 'ls /root' --become --become-user=root --become-method=su --ask-become-pass
    BECOME password:

测试执行没有问题，就可以移除有问题的配置文件：

    ansible all -m raw -a 'mv -iv /etc/sudoers.d/BAD_CONF /tmp' --become --become-user=root --become-method=su --ask-become-pass
    BECOME password:

# reference

<https://docs.ansible.com/ansible/latest/user_guide/become.html>

<br/>

本文标题 | [{{ page.title }}]({{ page.url }})
-------- |:--------
原始链接 | <{{ site.url }}{{ page.url }}>