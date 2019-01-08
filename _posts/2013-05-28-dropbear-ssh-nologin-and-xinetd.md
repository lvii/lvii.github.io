---
layout: post
title: dropbear /sbin/nologin 和 xinetd 服务模式遇到的问题
category: server
tags: [dropbear, ssh, xinetd]
---

在低配的 VPS 和 openwrt 路由器中可用 dropbear 替换 openssh 作为 ssh server 节省些资源

在设置帐号 shell 为 `/sbin/nologin` 以及 `xinetd` 模式管理 dropbear 服务时遇到些问题：

- 用户 shell 设为 `/sbin/nologin` 导致无法连接 ssh
- `xinetd` 服务模式下，自定义网络端口失败

## `/sbin/nologin` 导致无法连接 ssh

为了方便翻墙，创建一个独立的翻墙用户，和日常用户区分开，互不干扰

仅仅作 sock 代理，用户的 shell 设置为 `/sbin/nologin` 不需要登录，也保证了安全性

    $ sudo useradd -m -s /sbin/nologin <username>
    $ ssh-keygen -b 4096 -t rsa -N '' -C 'FUCKGFW' -f id_rsa.fuckgfw
    $ sudo cat id_rsa.fuckgfw.pub >> /home/<username>/.ssh/authorized_keys
    $ chmod 600 /home/<username>/.ssh/authorized_keys

但是后面登录的时候杯具了：

    $ ssh -vvv -NTfn -D 7070 fuckgfw@example.com
    debug1: Authentications that can continue: publickey
    debug3: start over, passed a different list publickey
    debug3: preferred publickey,keyboard-interactive,password
    debug3: authmethod_lookup publickey
    debug3: remaining preferred: keyboard-interactive,password
    debug3: authmethod_is_enabled publickey
    debug1: Next authentication method: publickey
    debug1: Offering RSA public key: /home/fuckgfw/.ssh/id_rsa
    debug3: send_pubkey_test
    debug2: we sent a publickey packet, wait for reply
    debug1: Authentications that can continue: publickey
    debug1: Trying private key: /home/fuckgfw/.ssh/id_dsa
    debug3: no such identity: /home/fuckgfw/.ssh/id_dsa
    debug1: Trying private key: /home/fuckgfw/.ssh/id_ecdsa
    debug3: no such identity: /home/fuckgfw/.ssh/id_ecdsa
    debug2: we did not send a packet, disable method
    debug1: No more authentication methods to try.
    Permission denied (publickey).

由于关闭了密码登录 `Permission denied (publickey).` 错误一般是指密钥有问题

检查了 `/home` 和 `~/.ssh` 目录相关的文件/夹 **权限** 都是正确的，那为啥杯具了。。。

`chsh` 将 shell 改回 `/bin/bash` 是可以正常连接的，那问题应该出在 `/sbin/nologin`

重新连接，在服务端查看 `/var/log/messages` 发现 ssh 连接被拒的信息：

    dropbear[24147]: Child connection from 116.120.72.101
    dropbear[24147]: User 'fuckgfw' has invalid shell, rejected
    dropbear[24147]: Exit before auth (user 'fuckgfw', 2 fails): Exited normally

日志提示 `invalid shell` 非法的 shell

google 到这篇文章：[dropbear dropbear[2543]: user 'Bhavdip' has invalid shell, rejected](http://blog.sina.com.cn/s/blog_6ab264c601011s8m.html)

文中说要将 `/sbin/nologin` 追加到 `/etc/shells` 文件。添加之后 ssh 正常连接

    dropbear[24667]: Child connection from 122.120.72.111:20086
    dropbear[24667]: Pubkey auth succeeded for 'fuckgfw' with key from 122.120.72.111:20086

`man 5 shells` 科普了一下 `/etc/shells` 文件的作用：验证用户 shell 的合法性

> `/etc/shells` is a text file which contains the **full pathnames** of valid login shells.
> This file is consulted by `chsh(1)` and available to be queried by other programs.
> Be aware that there are programs which consult this file to find out if a user is a normal user;
> for example, **FTP daemons traditionally disallow access to users with shells not included in this file**.

看来 ssh 客户端调试的错误信息指向，还是不是很明确的，服务端的日志还是蛮给力的！

## 使用 xinetd 接管 dropbear 服务

默认 dropbear 是以 **standalone 模式** 启动的 dropbear 守护进程会常驻内存，
改为 xinetd 监听端口，根据 ssh 连接请求来 **开启/关闭** dropbear 进程，可以节省些资源
ssh 服务端口自定义过，在 xinetd 启动 dropbear 时，**自定义端口** 时又遇到了问题:

    # cat /etc/xinetd.d/dropbear
    service dropbear
    {
        socket_type     = stream
        only_from       = 0.0.0.0
        wait            = no
        user            = root
        protocol        = tcp
        server          = /usr/sbin/dropbear
        server_args     = -i -s -w -K 60
        #type            = UNLISTED
        port            = 55555
        disable         = no
    }

配置的参数主要是:

option | about
------ | -----
`-i` | Start for inetd ( `xinetd` 服务模式)
`-w` | Disallow root logins
`-s` | Disable password logins
`-K <keepalive>` | `0` is never, default `0` in seconds

- 禁用密码登录
- 禁止 root 登录
- 设置 keepalive 连接保持时间

dropbear 在 **standalone** 服务模式下的 `-p` 参数可以配置端口

    -p [address:] port Listen on specified address and TCP port

但在 xinetd 下无效，需要由 xinetd 服务来接管对 **端口** 的监听

重启 xinetd 服务 xinetd 没有成功接管 dropbear 日志中报错了：

    xinetd[24563]: service/protocol combination not in /etc/services: dropbear/tcp
    xinetd[24563]: xinetd started with libwrap loadavg options compiled in.
    xinetd[24563]: Started working: 0 available services

错误提示 xinetd 无法解析 dropbear 对应的端口 xinetd 解析常用的服务端口有两种方法：

- `/etc/services` (默认)
- `/etc/xinetd.d/services` 服务配置中指定的 port 参数

`/etc/services` 中没有指定 dropbear 的自定义端口，所以 xinetd 没有成功接管 dropbear

但在 `/etc/xinetd.d/dropbear` 中自定义的端口 `port=55555` 木有没生效

**这篇文章**：[ubuntu 下使用 xinetd](http://phl.iteye.com/blog/1815778)

提到修改 xinetd 配置文件中的端口变量时，还需要指定 `type = UNLISTED`

`man xinetd.conf` 解释如下：

>   `type UNLISTED`  
>   if this is a service not listed in a standard system file (like
>   `/etc/rpc` for RPC services, or `/etc/services` for non-RPC services).

在配置文件中添加 `type = UNLISTED` 重启 xinetd 后成功接管 dropbear ：

    xinetd[24659]: xinetd started with libwrap loadavg options compiled in.
    xinetd[24659]: Started working: 1 available service

此时也可以查看到 xinetd 监听的端口信息：

    # netstat -lntp
    Active Internet connections (only servers)
    Proto Recv-Q Send-Q    Local Address    Foreign Address    State     PID/Program name
    tcp        0      0    0.0.0.0:55556    0.0.0.0:*          LISTEN    1806/sshd
    tcp        0      0    0.0.0.0:55555    0.0.0.0:*          LISTEN    24659/xinetd

到此就搞定了 xinetd 模式的 dropbear 配置


<br/>

本文标题 | [{{ page.title }}]({{ page.url }})
-------- |:--------
原始链接 | <{{ site.url }}{{ page.url }}>
