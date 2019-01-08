---
layout: post
title: /etc/hosts 指定 IP 访问 google ssl 加密搜索
category: web
tags: [google, ssl]
---

google 搜索被重定向太不爽了，要么翻墙代理，不然就土一点，指定 hosts

用代理要绕到代理服务器这个中介来访问，速度难免会有些慢

修改 hosts 直接指定访问的 IP 需要找的就是那个本地访问快些的 IP

现在 www.google.com 有了 https 的搜索，但是跳转到相关网页时，依然被和谐

然后 encrypted.google.com 该出场了，亲，这个可是全程走 ssl 的

## 简单点

最简单的方法就是 `ping www.google.com.hk` 获取 **香港** 那边服务器的 IP

    · ping www.google.com.hk
    PING www-wide.l.google.com (74.125.128.199) 56(84) bytes of data.

## 复杂点

[just-ping](http://www.just-ping.com) 网站可以获取全球各个地区 ping google 服务器返回的 IP 地址

搜索 encrypted.google.com 根据 ping 时间较快的筛选 IP 地址写入 hosts

现在好些发达国家 ping 的 IP 都已经是 `IPv6` 了，看来社会主义落后了

    $ grep encrypt /etc/hosts
    74.125.237.14 encrypted.google.com

可以使用多个 host 别名，什么新加坡，芝加哥，英格兰，南非的速度也可以

## 注意

1. hosts 文件中 **不要** 在域名前面添加 `https://` 否则不能访问
2. 修改完 hosts **第一次** 访问需要先在 firefox 使用 `https://host_ip` 验证下 CA 证书
3. 然后访问 `google.com.hk` 点击 **右下角** `google.com` 切到 **国际版** 获取对应 cookie

如果浏览器记住的是 **香港站** cookie 后面会跳转到 `https://encrypted.google.com.hk`

之后就可以使用 [Google SSL 加密搜索](https://encrypted.google.com) 了，从此麻麻再也不用担心你的搜索了

虽然麻烦了点，但总比搜索敏感词被重置好！

P.S. 像 gmail / google+ / ~~greader~~ / docs 都可以使用类似的，修改 host 曲线翻墙




<br/>

本文标题 | [{{ page.title }}]({{ page.url }})
-------- |:--------
原始链接 | <{{ site.url }}{{ page.url }}>
