---
layout: post
title: "Fedora 33 ssh RSA 密钥认证 Permission denied 失败问题"
category: system
tags: [ssh]
---

# WHAT

升级到 Fedora 33 后，发现 ssh 密钥认证登录远程服务器失败，提示 `Permission denied (publickey).`

`ssh -vvv user@10.16.2.99` 认证过程有识别到 `~/.ssh/id_rsa` 但是并未成功：

    debug1: Authentications that can continue: publickey
    debug3: start over, passed a different list publickey
    debug3: preferred publickey,keyboard-interactive,password
    debug3: authmethod_lookup publickey
    debug3: remaining preferred: keyboard-interactive,password
    debug3: authmethod_is_enabled publickey
    debug1: Next authentication method: publickey
    debug1: Offering public key: /home/user/.ssh/id_rsa RSA SHA256:... explicit agent
    debug1: send_pubkey_test: no mutual signature algorithm
    debug2: we did not send a packet, disable method
    debug1: No more authentication methods to try.
    user@10.16.2.99: Permission denied (publickey).

# WHY

Fedora 33 禁用 RSA 认证算法：

<https://fedoraproject.org/wiki/Changes/StrongCryptoSettings2>

可以使用下面命令调整 **全局** 认证规则：

    update-crypto-policies --set DEFAULT:FEDORA32
    update-crypto-policies --set LEGACY

# HOW

比较简单的方法就是在 `~/.ssh/config` 启用 `ssh-rsa` ：

    echo 'PubkeyAcceptedKeyTypes +ssh-rsa' >> ~/.ssh/config

# reference

<https://stackoverflow.com/questions/64640596/ssh-permission-denied-publickey-after-upgrade-fedora-33>

<https://bugzilla.redhat.com/show_bug.cgi?id=1881301>