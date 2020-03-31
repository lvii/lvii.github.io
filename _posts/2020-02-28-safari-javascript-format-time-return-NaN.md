---
layout: post
title: "Safari js Date() 格式化时间字符串返回 NaN 的不兼容问题"
category: code
tags: [javascript]
---

# WHAT

登录页面验证 token 过期时间的逻辑，在 Safari 出现死循环

调试发现判断 token 过期的格式化时间戳变量返回值为 `NaN`

变量 `expireTimeString` 为 `yyyy-MM-dd HH:mm:ss` 格式的时间字符串

Safari 浏览器 `Date()` 格式化为 Unix 时间戳返回的是 **Not-A-Number** 字符串 `NaN` ：

``` javascript
const expireSeconds = new Date(expireTimeString).getTime();
```

# WHY

Safari 浏览器不支持 `Date('yyyy-MM-dd HH:mm:ss')` 格式的时间字符串

[JavaScript new Date() Returning NaN in IE or Invalid Date in Safari 2011-02-08](http://biostall.com/javascript-new-date-returning-nan-in-ie-or-invalid-date-in-safari/)

# HOW

需要在 `yyyy-MM-dd HH:mm:ss` **日期** 后面添加 `T` 标记，转换为 `yyyy-MM-ddTHH:mm:ss` 格式：

``` javascript
const expireSeconds = new Date(expireTimeString.replace(/\s/, 'T')).getTime();
```

如果日期是 UTC 时间，还需要在末尾追加 `Z` 标记，转换为 `yyyy-MM-ddTHH:mm:ssZ` 格式：

``` javascript
const expireSeconds = new Date(expireTimeString.replace(/\s/, 'T')+'Z').getTime();
```

# reference

[Safari Javascript Date() NaN Issue (yyyy-MM-dd HH:mm:ss) 2014-02-19](https://stackoverflow.com/questions/21883699/safari-javascript-date-nan-issue-yyyy-mm-dd-hhmmss)

<https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Date>

<br/>

本文标题 | [{{ page.title }}]({{ page.url }})
-------- |:--------
原始链接 | <{{ site.url }}{{ page.url }}>