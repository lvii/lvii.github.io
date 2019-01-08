---
layout: post
title: SSH 远程端口转发
category: system
tags: [ssh]
---

ssh 的远程端口转发，已有不少文章科普，可是自己搞了才知道，原来是有坑的啊

## 端口转发 != 隧道

有些文章，直接让这两个术语搞基了 `-_-#`

- **端口转发** `port forwarding` 只需要与服务器对应的端口建立 TCP 连接即可
- **隧道** `tunnel` 可不一样，需要依赖虚拟网卡 `tun` 设备来通讯

下面是 `man ssh` 对 ssh 隧道选项 `-w` 的解释：

    -w local_tun[:remote_tun]
        Requests tunnel device forwarding with the specified tun(4) devices
        between the client (local_tun) and the server (remote_tun.)

wikipedia 科普 TUN 设备：[「Linux 内核中的虚拟网络设备 TUN 与 TAP」](http://zh.wikipedia.org/zh/TUN与TAP)

> TUN 模拟了网络层设备，操作第三层数据包比如 IP 数据封包

## 远程端口转发

跑题了，来看看 `man ssh` 对 **远程端口转发** 选项 `-R` 的描述：

    -R [bind_address:]port:host:hostport
        Specifies that the given port on the remote (server) host is to be forwarded to the given host and port on the local side. This works by allocating a socket to listen to port on the remote side, and whenever a connection is made to this port, the connection is forwarded over the secure channel, and a connection is made to host port hostport from the local machine.  ...

        By default, the listening socket on the server will be bound to the loopback interface only.  This may be overridden by specifying a bind_address.  An empty bind_address, or the address ‘*’, indicates that the remote socket should listen on all interfaces.  Specifying a remote bind_address will only succeed if the server's GatewayPorts option is enabled (see sshd_config(5)).

参数有些多，有点乱，引用这篇文章 [「SSH port forwarding visualized」][1] 的图片，

图文并茂的解释下 man 手册中各个参数和各个角色的对应关系，无图无真相啊：

![ssh-remote-port-forwarding.png][2]

### 实例场景

厂里线上 **内网** 环境有一台机器，无法在办公网直接访问，需要拨 VPN ，登录跳板机各种 。。。

但是这台 **内网** 机器，可以连接 **公网** 的 VPS ，办公环境也可以访问 VPS

那就将 VPS 作为中转，通过转口转，实现在办公网 **"直接"** 访问线上 **内网** 机器

图中各个角色的对应关系如下：

角色        | 描述
        --- | ---
APP         | 办公室的笔记本
sshd server | **公网** VPS ( 映射端口：`port 2222` )
ssh client  | **内网** 机器
APPSRV      | 与 **内网** ssh client 是同一台机器 ( 目的端口： `hostport 22` )

远程端口转发命令要在 **内网** 机器上执行，即从 **内网** 连接 **公网** VPS ：

    (ssh clinet) # ssh -fnNT -R 2222:localhost:22 mantou.me

命令行选项解释：

选项 | 解释
---- | ----
`-f` | 将 ssh 转到 **后台运行**，即认证之后 ssh 自动以后台运行，不在输出信息
`-n` | 将 stdio 重定向到 `/dev/null` 与 `-f` 配合使用
`-N` | 不执行脚本或命令，即通知 sshd 不运行设定的 shell 通常与 `-f` 连用
`-T` | 不分配 TTY 只做代理用
`-q` | 安静模式，不输出 **错误/警告** 信息

使用 `lsof` 可以看到是否成功建立连接：

    (ssh clinet) # lsof -p <ssh_remote_forwarding_PID>
    COMMAND  PID USER   FD   TYPE   DEVICE SIZE    NODE NAME
    ssh     1912 root    3u  IPv4    43896         TCP  localhost:54478->199.108.255.25:22 (ESTABLISHED)

成功建立连接后，在 VPS 上面可以监听到 `2222` 端口：

    (sshd server) $ netstat -lntp
    Proto Recv-Q Send-Q Local Address     Foreign Address    State     PID/Program
    tcp        0      0 127.0.0.1:2222    0.0.0.0:*          LISTEN    2169/sshd:
                        ^^^^^^^^^
                        默认仅监听 VPS 本地端口 ( sshd 启用 GatewayPorts 选项才可监听 0.0.0.0 )

    (sshd server) # lsof -p 2169
    COMMAND  PID USER   FD   TYPE   DEVICE SIZE    NODE NAME
    sshd    2169 root    3u  IPv4    12685  0t0    TCP  199.108.255.25:22->125.124.11.28:54478 (ESTABLISHED)

此时就可以在 VPS 上访问 **内网** 机器：

    $ ssh -p 2222 user@localhost

## 注意（坑）

- 需要 root 权限，不然会提示权限错误：`Privileged ports can only be forwarded by root`
- 不能使用 VPS (sshd server) 已占用的 `22` 端口，用作端口转发。选用 **未使用** 端口(比如 `2222`)  
  否则就杯具了：`Warning: remote port forwarding failed for listen port 22`

### GatewayPorts

    -R [bind_address:]port:host:hostport

`bind_address` 参数 **默认为空**，等价于`*:port:host:hostport`
并不意味着任何机器，都可以通过 VPS 来访问 **内网** 机器。建立连接后，只能在 VPS ( sshd server )  **本地** ( 默认监听 `localhost` )
访问 「内网」 机器。要在办公网的笔记本上通过 VPS 映射的端口来访问 **内网** 机器，需要启用 VPS sshd 的 `GatewayPorts` 参数，
允许任意请求地址 ( 监听 `0.0.0.0` ) 通过转发的端口访问内网机器。

下面是 `man sshd_config` 手册对 `GatewayPorts`* 选项的解释：

    GatewayPorts
        Specifies whether remote hosts are allowed to connect to ports forwarded for the client.  By default, sshd(8) binds remote port forwardings to the loopback address.  This prevents other remote hosts from connecting to forwarded ports.  GatewayPorts can be used to specify that sshd should allow remote port forwardings to bind to non-loopback addresses, thus allowing other hosts to connect.  The argument may be “no” to force remote port forwardings to be available to the local host only, “yes” to force remote port forwardings to bind to the wildcard address, or “clientspecified” to allow the client to select the address to which the forwarding is bound.  The default is “no”.

启用 `GatewayPorts` 之后，才可以从办公室的笔记本访问 VPS `2222` 端口，连接 **内网** 的机器：

    (APP) $ ssh -p 2222 user@example.com

远程端口转发的应用场景抽象一下，对于没有 **公网 IP** 、在 **NAT 隔离** 下的内网的机器（还有虚拟机），都可以
使用 ssh 远程端口转发简单快捷的访问机器。若是内网 Linux 遇到问题，就可以邀请好友来远程协助什么的，，，

## 安全

通过远程端口转发来连接 **内网** 服务端口，最好指定 `bind_address` **安全第一**

[1]: http://www.dirk-loss.de/ssh-port-forwarding.htm
[2]: http://media-cache-ec0.pinimg.com/originals/9f/37/b4/9f37b45ae14c4b58ac3aafd960272a62.jpg


<br/>

本文标题 | [{{ page.title }}]({{ page.url }})
-------- |:--------
原始链接 | <{{ site.url }}{{ page.url }}>
