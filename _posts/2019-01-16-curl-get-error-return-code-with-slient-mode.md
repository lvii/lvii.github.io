---
layout: post
title: "使用 curl -sS 请求失败时返回失败的返回值"
category: soft
tags: [curl]
---

# WHAT

平时一直习惯使用 `curl` 命令的 `-sSL` 组合参数 **静默下载** 文件。以为这就是 **良药** ... 结果：

    $ curl -sSL http://IP/ok && echo -e "\nNOTE: curl return code=$?"
    TEXT FILE

    NOTE: curl return code=0

    $ curl -sSL http://IP/error && echo -e "\nNOTE: curl return code=$?"
    <html>
    <head><title>404 Not Found</title></head>
    <body bgcolor="white">
    <center><h1>404 Not Found</h1></center>
    <hr><center>nginx</center>
    </body>
    </html>

    NOTE: curl return code=0

请求返回 `404` **失败** 时，命令对应的 **返回值** 却依然是 `0` 这还了得 - -'

# HOW

查看 `man curl` 发现真正的 **良药** 是 `-f, --fail` 参数 `-sS` 只是安慰剂 ...

    $ curl -f -sSL http://IP/error || echo -e "\nNOTE: curl return code=$?"
    curl: (22) The requested URL returned error: 404 Not Found

    NOTE: curl return code=22

    $ curl -f -sSL http://IP/error 2>/dev/null || echo -e "\nNOTE: curl return code=$?"

    NOTE: curl return code=22

shell 脚本里 `curl` 下载的 **返回值** 还是蛮重要的，得赶紧补上 `-f` 参数

<br/>

本文标题 | [{{ page.title }}]({{ page.url }})
-------- |:--------
原始链接 | <{{ site.url }}{{ page.url }}>
