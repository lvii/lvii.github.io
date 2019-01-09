---
layout: post
title: "MacOS 自然码双拼输入法 —— 鼠鬚管"
category: system
tags: [rime, squirrel, macos]
---

# WHY

公元 9102 年 MacOS 内置中文输入法依然不支持 **最基础** 的双拼码表：**自然码** - -'

# WHAT

[RIME、「中州韻」、「小狼毫」和「鼠鬚管」来历](https://rime.im/blog/2016/04/14/qna-in-mtvu/)

# HOW

沉寂 4 年后 RIME 中文输入法平台 MacOS 下的实现 **鼠鬚管** 更新至 `0.10.0` (2019-01-01)

 决定重新回归小巧的 **鼠鬚管** 输入法

<https://rime.im/release/squirrel/>

> 精簡安裝包預裝的輸入方案，更多方案可由 [東風破](https://github.com/rime/plum) 取得

新版本 `0.10.0` 默认只有 **朙月拼音** 输入法，其他第三方要用 [東風破](https://github.com/rime/plum) 额外安装

下载安装后 **鼠鬚管** 的 **朙月拼音** 输入法是 **繁体中文**

使用 `` Ctrl + ` `` 快捷键即可切换至 **简体中文** ：

![img](https://i.imgur.com/BKlDSQn.png)

## `double_pinyin.schema.yaml`

[https://github.com/rime/home/wiki/UserGuide#雙拼](https://github.com/rime/home/wiki/UserGuide#%E9%9B%99%E6%8B%BC)

> 雙拼與【朙月拼音】、語句流輸入方案 **共用** 一份 **碼表** 和 **用戶詞典**。

RIME 的双拼输入方案：<https://github.com/rime/rime-double-pinyin>

默认即是 **自然码** 输入方案：

<https://github.com/rime/rime-double-pinyin/blob/master/double_pinyin.schema.yaml>

RIME 用户配置目录位于 `~/Library/Rime/` 相关的 YAML 配置文件：

配置文件 | 关于
:------- | :---
`default.custom.yaml` | 配置 **可用** 输入法列表 <sup>[1](https://github.com/rime/home/wiki/RimeWithSchemata#rime-中的數據文件分佈及作用)</sup>
`squirrel.custom.yaml` | 配置 **候选框** 配色方案 <sup>[2](https://github.com/rime/home/wiki/CustomizationGuide#鼠鬚管外觀與鍵盤設定)</sup>
`double_pinyin.schema.yaml` | 配置 **双拼自然码** 码表 <sup>[3](https://github.com/rime/home/wiki/RimeWithSchemata#三最高武藝)</sup>

下载配置：

    $ cd ~/Library/Rime/

    $ yaml_path=https://raw.githubusercontent.com/rime/rime-double-pinyin/master/double_pinyin.schema.yaml

    $ curl -L $yaml_path -o double_pinyin.schema.yaml
      % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                     Dload  Upload   Total   Spent    Left  Speed
    100  3074  100  3074    0     0   1566      0  0:00:01  0:00:01 --:--:--  1567

下载的配置依赖 **筆畫** 输入法，需要注释相关配置，并设置默认为 **简体中文** ：

    $ cp -iv double_pinyin.schema.yaml{,.ori}
    double_pinyin.schema.yaml -> double_pinyin.schema.yaml.ori

    $ diff -y double_pinyin.schema.yaml double_pinyin.schema.yaml.ori

      # dependencies:                         |   dependencies:
      #   - stroke                            |     - stroke

        reset: 1                              <
        states: [ 漢字, 汉字 ]                      states: [ 漢字, 汉字 ]

    # reverse_lookup:                         | reverse_lookup:
    #   dictionary: stroke                    |   dictionary: stroke
    #   enable_completion: true               |   enable_completion: true
    #   prefix: "`"                           |   prefix: "`"
    #   suffix: "'"                           |   suffix: "'"
    #   tips: 〔筆畫〕                        |   tips: 〔筆畫〕
    #   preedit_format:                       |   preedit_format:
    #     - xlit/hspnz/一丨丿丶乙/            |     - xlit/hspnz/一丨丿丶乙/
    #   comment_format:                       |   comment_format:
    #     - xform/([nl])v/$1ü/                |     - xform/([nl])v/$1ü/

[【智能ABC雙拼】輸入方案示例](https://github.com/rime/home/wiki/RimeWithSchemata#三最高武藝)

## `default.custom.yaml`

然后修改 `default.custom.yaml` 配置的可用输入法列表：

    $ cat default.custom.yaml
    patch:
      schema_list:
        - schema: double_pinyin        # 自然码双拼
        - schema: luna_pinyin_simp     # 简体全拼

## `squirrel.custom.yaml`

修改 **候选框** 样式：

    $ cat squirrel.custom.yaml
    patch:
      style:
        color_scheme: google                    # 方案命名，不能有空格
        horizontal: true
        inline_preedit: true                    # 单行显示，false双行显示
        candidate_format: "%c\u2005%@"          # 用 1/6 em 空格 U+2005 来控制编号 %c 和候选词 %@ 前后的空间
        corner_radius: 3                        # 候选条圆角
        hilited_corner_radius: 3                # 高亮圆角
        border_height: 4                        # 窗口边界高度，大于圆角半径才生效
        border_width: 4                         # 窗口边界宽度，大于圆角半径才生效
        border_color_width: 0
        font_face: "PingFangSC"                 # 候选词字体
        font_point: 16                          # 候选字词大小
        label_font_point: 14

# reference

Squirrel 内置 **候选框** 样式：<https://gist.github.com/wd/26e0ce3751e2b8da64a3300dc7c0f926>

![img](https://user-images.githubusercontent.com/58177/50718674-0c2b1180-10cd-11e9-85e4-b0c2cb185f21.png)

<https://github.com/rime/home/wiki/CustomizationGuide#一例定製小狼毫配色方案>

![img](http://i.imgur.com/hSty6cB.png)

[我的 Rime 2017-06-17](https://blog.dwx.io/my-rime/)

# TODO

[「Rime 增强计划」—— 让小狼毫更好用の BetterRime 增强包です](http://tieba.baidu.com/p/4125987751)

<https://github.com/xiaoTaoist/rime-dict>

[Rime 詞庫](https://github.com/rime-aca/dictionaries)

<br/>

本文标题 | [{{ page.title }}]({{ page.url }})
-------- |:--------
原始链接 | <{{ site.url }}{{ page.url }}>
