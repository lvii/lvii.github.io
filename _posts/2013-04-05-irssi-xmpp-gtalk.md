---
layout: post
title: irssi-xmpp 连接 gtalk 和 xmpp 服务器
category: soft
tags: [irssi, xmpp, plugin, irc]
---

折腾一下 `irssi-xmpp` 插件登录 gtalk 和 xmpp 服务

快速入门参考手册：

    $ sudo emerge -avj irssi-xmpp
    $ bzcat /usr/share/doc/irssi-xmpp-0.52/STARTUP.bz2|less

## 登录 gtalk

在 irssi status 窗口下，手动执行命令方式来连接 gtalk 服务器：

    /load xmpp

    09:36          * | Loaded module xmpp/core
    09:36          * | Loaded module xmpp/text
    09:36          * | Loaded module xmpp/fe

    /xmppconnect -host talk.google.com <username>@gmail.com <password>

    09:44          * | Looking up talk.google.com
    09:44          * | Connecting to talk.google.com port 5222
    09:44  Using STARTTLS encryption.
    09:44  Authenticated successfully.
    09:44  Requesting the roster.
    09:44          * | Connection to talk.google.com established


连接成功后，使用 `/roster` 命令查看 gtalk 在线好友列表 `/msg <gmail_or_nick>` 来聊天了

## 配置登录

把服务器信息写入 irssi 配置文件，启动后可以使用 `/connect <gtalk_server>` 连接 gtalk

- irssi 启动自动加载 xmpp 模块：`echo 'load' >> ~/.irssi/startup`
- irssi 读入定义的 server 信息，后面使用 `/connect <gtalk_server>` 连接

详细配置参考这篇帖子：[How To use Gtalk the Minimalist way](http://crunchbang.org/forums/viewtopic.php?id=17703)

irssi-xmpp 插件常用的有 2 个命令：

    /roster     添加(add)，显示(full)，修改(name)，分组(group)，删除(remove) xmpp 联系人操作
    /presence   对添加好友的 subscription 邀请进行 accept, deny 操作

其他的日常命令如 /whois /msg 以及 /set 设置 xmpp 插件选项，详细用法参考手册：

    $ bzcat /usr/share/doc/irssi-xmpp-0.52/GENERAL.bz2|less

## 好友添加

当别人发送添加好友申请 (subscription) 时，会在 irssi 控制台提示下面消息：

    13:27 <username>@gmail.com: wants to subscribe to your presence  (accept or deny?)

    /roster full

    13:28  ROSTER:<username>@xmpp.jp irssi-xmpp(0)
    13:28   | UNKOWN GROUP |:
    13:28     (+) vim-cn<username>@vim-cn.com bot(30): 主题：Vim、Linux、编程。勿水，谢谢合作
    13:28     (+) 用户<username>@gmail.com kde-telepa5FCEA6FB(0)
    13:28  End of ROSTER

接收添加好友申请：

    /persence accept<username>@gmail.com

`/roster full` 查看好友已出现，需要将好友添加至联系人列表：

    /roster full

    13:30  ROSTER:<username>@xmpp.jp irssi-xmpp(0)
    13:30   | UNKOWN GROUP |:
    13:30     (+) vim-cn<username>@vim-cn.com bot(30): 主题：Vim、Linux、编程。勿水，谢谢合作
    13:30     (+) 用户<username>@gmail.com kde-telepa5FCEA6FB(0)
    13:30     (-)<username>@gmail.com  (subscription: from)
    13:30  End of ROSTER

    /roster add<username>@gmail.com
    13:31 <username>@gmail.com: wants you to see his/her presence

    /roster full
    13:32  ROSTER:<username>@xmpp.jp irssi-xmpp(0)
    13:32   | UNKOWN GROUP |:
    13:32     (+)<username>@gmail.com Talk.v304CDF876D4(24)gmail.B49D8CB0(24)
    13:32     (+) vim-cn<username>@vim-cn.com bot(30): 主题：Vim、Linux、编程。勿水，谢谢合作
    13:32     (+) 用户<username>@gmail.com kde-telepa5FCEA6FB(0)

这样才算完整的接受好友添加

添加好友和上面类似，发送添加好友申请 `/PRESENCE SUBSCRIBE <jid>`

对方接受后，再使用 `/roster` 命令修改联系人信息

## 配置选项

xmpp 设置选项可以在 irssi 控制台使用 `/set` 命令查看当先配置

    /set xmpp_roster_show_offline

    22:10  [xmpp_roster]
    22:10  xmpp_roster_show_offline = OFF

把相应的设置写入 irssi 配置文件时，配置选项会有几个不同的 **section** ：

    fe-common/xmpp
    xmpp/core

但文档木有详细介绍，我也是 [参考别人的配置][1] 加了几个自己配置而已

改好配置文件使用 `/reload` 重新加载，如果有问题会有提示的

还可以在控制台用 `/set` 手动查看配置是否生效

再土一点就是备份 irssi 配置文件，设置好对应选项后 `/save` 生成一份新的配置

其中关于 SSL 配置文件无须指定 `use_ssl` 选项，默认连接会使用 `STARTTLS`

而且 `use_ssl='yes'` 使用的是 old SSL 格式已经被 deprecated 了，不推荐使用

## xmpp 服务

虽然可以用上 gtalk 但还是有几个问题：

1. gtalk 有时候不是很稳定，墙，会不时把它搞掉线，连接会有些不稳定
2. google 某天或许会像干掉 google reader 那样宰掉 gtalk

所以需要一个开放的平台支持，可以申请第三方 xmpp 服务

可申请服务列表如下：[public XMPP services](http://xmpp.net/)

根据第三方 xmpp 服务支持的 `<JID>` 格式，申请帐号 `/XMPPREGISTER -host <xmpp_sever> <JID> <password`

申请完到帐号后，连接方式和 gtalk 一样，可用 `/xmppconnect` 手动连接或写入配置文件

后来从 `xmpp.jp` 转到 `xmpp.vim-cn.com` 使用 irssi-xmpp 注册失败:

1. `irssi-xmpp` 不支持解析 DNS SRV 记录
2. vim-cn 用的自签 SSL 证书 `irssi-xmpp` 没有参数可以忽略自签证书问题

改用 telepathy 注册，勾选 **忽略 SSL 错误** 选项才行，注册后可以使用 irssi-xmpp 登录

## hack

xmpp 接受的 **回复** 消息，对应的是 irssi 的 **私信** `notify.pl` 插件会把 xmpp 群 **每个回复** 都进行提示。

需要改一下 [notify.pl](https://github.com/lvii/dotfiles/blob/master/irssi/scripts/notify.pl) 对 xmpp 群进行过滤，普通好友不影响

## 优点

不用像 bitlbee 那样需要启动本地服务，然后再添加 gtalk 的 irc 服务，还要再配置 irssi

集成进 irssi 这样 irc 和 xmpp 就是一家人了，可以在终端下面一起查看了，不用多个窗口来回切换了

## 缺点

irssi 聊天开启的窗口名称使用的是 `<JID>/resource_name` 而不能自定义为 `nickname`
[TODO](http://cybione.org/cgi-bin/cvsweb/~checkout~/irssi-xmpp/TODO?rev=1.12&content-type=text/plain&cvsroot=irssi-xmpp)
中说是插件不好对 irssi 的回复 signal 进行处理，只能重新模拟一个 signal 来替换。
而且搞起来还很烦，就这个问题问了开发者，作者貌似不想搞这个，但他说会接收 irssi 高手的 patch

## 无图无真相

![irss xmpp](http://fc02.deviantart.net/fs71/f/2013/100/7/a/irssi_xmpp_by_57lvii-d61373j.png)


<br/>

本文标题 | [{{ page.title }}]({{ page.url }})
-------- |:--------
原始链接 | <{{ site.url }}{{ page.url }}>
