---
layout: post
title: centos 6 搭建 cacti 监控
category: server
tags: [cacti, monitor]
---

cacti 就是把 snmp 收集的监控信息交给 `rrdtool` 画出来，还保存到数据库，可以查看历史数据，因此还是有些依赖的：

- `httpd`
- `php`
- `php-mysql`
- `php-snmp`
- `php-ldap` (when using LDAP authentication)
- `php-xml`
- `mysql`
- `mysql-server`
- `net-snmp` (depending on the distro, `net-snmp-utils` may be required)
- `crond` (`cron`, `cronie` or the like)

cacti 官方手册有提供安装教程：<http://www.cacti.net/downloads/docs/html/>

后面安装好 cacti 软件包后，里面也内置了该离线文档。
centos 官方源中没有 cacti 。从 [「第三方 EPEL 软件源」][1] 安装

[1]: https://fedoraproject.org/wiki/EPEL/zh-cn#.E6.88.91.E6.80.8E.E6.A0.B7.E8.8E.B7.E5.8F.96_EPEL_.E7.9A.84.E8.BD.AF.E4.BB.B6.E5.8C.85.3F

    $ sudo yun instal -y cacti mysql-server net-snmp-utils

如果只安装 cacti 依赖软件不会装全，还需要手动安装 `mysql-server` 和 `net-snmp-utils` 依赖

安装好 cacti 后，可以参考离线文档，开始配置 cacti ：

    $ rpm -ql cacti|grep docs
    /usr/share/doc/cacti-0.8.8a/docs/txt/manual.txt

官方手册中有些配置针对 centos 发行版会有些不同，下面是我配置时发现的：

## PHP 模块路径

PHP 的扩展目录，使用 centos 默认的配置即可，只需额外配置时区即可

    ;extension_dir = /etc/php.d      ## cacti 手册
    ;extension_dir = "./"            ## centos 是注释的，默认是 OK 的
    date.timezone = Asia/Shanghai

**不要** 取消 `extension_dir` 注释，不然后面启动 cacti web 服务时 php 扩展路径有问题，会显示空白页面

`/etc/httpd/logs/error_log` 日志中会报错：

    PHP Startup: Unable to load dynamic library '/etc/php.d/curl.so'
    PHP Startup: Unable to load dynamic library '/etc/php.d/fileinfo.so'
    PHP Startup: Unable to load dynamic library '/etc/php.d/json.so'
    PHP Startup: Unable to load dynamic library '/etc/php.d/mysql.so'
    PHP Startup: Unable to load dynamic library '/etc/php.d/mysqli.so'
    PHP Startup: Unable to load dynamic library '/etc/php.d/pdo.so'
    PHP Startup: Unable to load dynamic library '/etc/php.d/pdo_mysql.so'
    PHP Startup: Unable to load dynamic library '/etc/php.d/pdo_sqlite.so'
    PHP Startup: Unable to load dynamic library '/etc/php.d/phar.so'
    PHP Startup: Unable to load dynamic library '/etc/php.d/snmp.so'
    PHP Startup: Unable to load dynamic library '/etc/php.d/sqlite3.so'
    PHP Startup: Unable to load dynamic library '/etc/php.d/zip.so'

centos 中 php 的模块都放在 `/usr/lib64/php/modules/` 路径

    $ ll /usr/lib64/php/modules/
    total 2.7M
    -rwxr-xr-x 1 root root  65K | 2013-02-22 | curl.so*
    -rwxr-xr-x 1 root root 1.8M | 2013-02-22 | fileinfo.so*
    -rwxr-xr-x 1 root root  36K | 2013-02-22 | json.so*
    -rwxr-xr-x 1 root root 133K | 2013-02-22 | mysqli.so*
    -rwxr-xr-x 1 root root  54K | 2013-02-22 | mysql.so*
    -rwxr-xr-x 1 root root  30K | 2013-02-22 | pdo_mysql.so*
    -rwxr-xr-x 1 root root 101K | 2013-02-22 | pdo.so*
    -rwxr-xr-x 1 root root  25K | 2013-02-22 | pdo_sqlite.so*
    -rwxr-xr-x 1 root root 256K | 2013-02-22 | phar.so*
    -rwxr-xr-x 1 root root  34K | 2013-02-22 | snmp.so*
    -rwxr-xr-x 1 root root  44K | 2013-02-22 | sqlite3.so*
    -rwxr-xr-x 1 root root  82K | 2013-02-22 | zip.so*

## apache 访问权限

cacti 的 `/etc/httpd/conf.d/cacti.conf` **默认** 访问权限是只让 `localhost` 连接，其他都拒绝：

    <Directory /usr/share/cacti/>
            <IfModule mod_authz_core.c>
                    # httpd 2.4
                    Require host localhost
            </IfModule>
            <IfModule !mod_authz_core.c>
                    ## 默认权限
                    # httpd 2.2
                    #Order deny,allow
                    #Deny from all
                    #Allow from localhost
                    ## 开放访问权限
                    Order Deny,Allow
                    Deny from none
                    Allow from all
            </IfModule>
    </Directory>


其他的配置都安装官方手册中来来即可。下面是两篇针对 centos 的配置教程，第二篇有配置之后的初始化图文简介：

- [[HOWTO] Cacti Install Guide CentOS 6 - 64](http://forums.cacti.net/viewtopic.php?f=6&t=49363)
- [How to install cacti on centos 6](http://www.krizna.com/centos/install-cacti-on-centos-6/)

配置下来，想了想 cacti 也算是流行的监控了，官方文档没有对主流的发行版自适应
centos 发行版也没有完善的文档，只能网上东奔西顾，为了配个服务遇到些，绊脚石 `-_-#`

## 后续遭遇

配置好后，添加新的监控机器后遇到了两个问题：

- 添加的虚拟机状态是 Device Status = Unknown
- 添加的设备再 graphs 页面左侧的 default tree 树中没找到

在 cacti 官方论坛中找到了解决方法：

- [Device Status = Unknown](http://forums.cacti.net/about21559.html&highlight=)
- [New graph not put under default tree](http://forums.cacti.net/viewtopic.php?f=21&t=42628)

设备状态 Unknown 是因为默认 crontab 每隔 5 分钟刷新一次，手工刷新就可以成功识别

    # /usr/bin/php /usr/share/cacti/poller.php

关于在 graphs 中 defautl tree 中找不到新加的节点，需要手动将机器添加到 default tree 中
还是看论坛的图文教程吧。。。还是要点点点点的，不过好像 cacti 有套 php 的命令行，，，
感觉还是很高端的样子。。。









<br/>

本文标题 | [{{ page.title }}]({{ page.url }})
-------- |:--------
原始链接 | <{{ site.url }}{{ page.url }}>
