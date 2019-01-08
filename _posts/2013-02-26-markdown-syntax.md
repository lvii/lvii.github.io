---
layout: post
title: markdown 语法
category: code
tags: [jekyll, github]
comments: false
---

- [markdown cheatsheet at github](https://github.com/adam-p/markdown-here/wiki/Markdown-Cheatsheet)
- [live demo 在线编辑预览](http://www.crypti.cc/markdown-here/livedemo.html)
- [GitHub Flavored Markdown](http://github.github.com/github-flavored-markdown)
- [markdown syntax at wikipedia](http://en.wikipedia.org/wiki/Markdown)
- [markdown 中文](http://wowubuntu.com/markdown)

atx 格式标题 C-c C-t 1...6 分别对应 html 的 `<h1>...<h6>` 标记

    # 标题一 #
    ## 标题二 ##
    ### 标题三 ###
    #### 标题四 ####
    ##### 标题五 #####
    ###### 标题六 ######

# 标题一 #
## 标题二 ##
### 标题三 ###
#### 标题四 ####
##### 标题五 #####
###### 标题六 ######

setext 格式标题 **标题一** 和 **标题二**


    标题一 title C-c C-t t
    ======================
    标题二 section C-c C-t s
    ------------------------

标题一 title C-c C-t t
======================
标题二 section C-c C-t s
------------------------

文本段落
----------
1. 段落是由 **一行** 或连续的 **多行** 的句子组成  
   即 **行内断行** 并不会在转换为 html 真正的换行符 `<br/>`  
   若想转换为 html 真正的换行，要在 **行尾** 增加 2 个以上的 **空格**
2. 段落之间的分隔，至少需要 **一行** 连续的 **空行** 来分隔
3. 文本行行首不能有 **空格** 或 **TAB** 缩进

Here's a line end with 2 more space  
This line is separated from the one above by two newlines, so it will be a *separate paragraph*.

This line is also a separate paragraph, but...[生成 html 后，两行是连接在一起的]
This line is only separated by a single newline, so it's a separate line in the *same paragraph*.

文本样式
----------
- C-c C-p b **粗体** `**粗体**`
- C-c C-p i *斜体* `*斜体*`
- C-c -     分割线 3 个以上连续的 `* - _` 字符
* * * * *
- 删除线 `~~line~~` 是否 markdown 标准语法 (?)  ~~删除线~~

列表 list
==========
1. 有序列表，使用 **数字**  作为前缀，不一定是顺序的，md 会自动解析
2. 无须列表
 - 前缀1：减号 `-`
 + 前缀2：加号 `+`
 * 前缀3：囧号 `#`
3. 列表换行
 - 对于列表项，如果文本行很长，编辑时可以换行，只要在行尾添加大于 1 个空格或 TAB  
   换行后可以缩进，最多 3 个空格，也可不缩进  
   本行列表项太长，要进行换行编辑，换行后不用缩进，不过缩进是为了美观 [编辑换行无效]
美观啊 ……
4. 如果有序列表项内容重又类似有序列表项格式：  
   `2013.02.05` 要用 `2013\.02\.05` 反斜杠来转义

引用 blockquotes C-c C-s b
--------------------------
引用行用 `>` 前缀

> 1. 引用中可以嵌套 markdown 语法：(github 无效)
> ## 内嵌标题 ##
> * 内嵌列表1
> * 内嵌列表2
> 2. 引用也可以嵌套引用
> > 要被嵌套的引用，嘿哈

代码块
----------
- 语法高亮 or monospace 字体 `标记` 使用 **反引号**  
  如果在代码中有用到 `` ` `` 反引号，可以用多个 **反引号**
- 如果在列表中引用 **代码块** 需要缩进 2倍，即缩进 8 个空格或 2 个 TAB  
        ``There is a literal backtick (`) here.``
- 在代码引用的开始和结束标记添加空格，就可以在代码起始位置使用 `` ` `` 反引号
A single backtick in a code span: `` ` ``
A backtick-delimited string in a code span: `` `foo` ``

示例
----------

    echo 'hello world 1级缩进'

- 用 4 个空格或 1 个 TAB 缩进  

        echo 'hello world 2级缩进，github 无效'
    echo 'hello world 1级缩进'

- 使用 `` ` `` 标记包裹


```
echo 'hello_world'
```

    ```
    echo 'hello_world 缩进过'
    ```

链接
----------
1. 行内链接 C-c C-a l

   `[link text](link.address.here)`

[Markdown](http://en.wikipedia.com/wiki/Markdown)

2. 行外链接 C-c C-a r  
   方便在文中多个地方引用相同的链接，集中管理，文本内容查看也整洁

```
    [link_name][link_id]
    [link_id]: http://link.address.here "注释: 要加 http:// 不然会解析为本地路径"
```

    [link_name][link_id]
    [link_id]: http://link.address.here "注释: 要加 http:// 不然会解析为本地路径"

[google][1]
[1]: http://google.com "google"
3. 相对路径，引用本地的 URL
`See my [About](/about/) page for details.`

图片
----------
- 和链接模式类似，只要在前面添加一个 `!` 叹号即可

```
    ![inline text](/path/to/img.jpg "Optional title")
    ![outline text][id]
    [id]: url/to/image  "Optional title attribute"
```

内嵌 html
---------
在 markdown 中可以直接使用 HTML 标签编辑 `<div> <table> <pre> <p>`

1. 要在 html 标记的前后面加空行，用来和普通 markdown 内容区分
2. html 段的起始标签不要 **缩进**，避免 markdown 解析为代码块
3. 实践发现上面两条 emacs 生成的预览中，不严格

```
<table>
    <tr>
        <td>这是个内嵌的 HTML 表格</td>
    </tr>
</table>
```

<table>
    <tr>
        <td>这是个内嵌的 HTML 表格</td>
    </tr>
</table>



<br/>

本文标题 | [{{ page.title }}]({{ page.url }})
-------- |:--------
原始链接 | <{{ site.url }}{{ page.url }}>
