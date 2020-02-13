---
layout: post
title: "python 解析 CSV 文件忽略逗号分隔的空数据行"
category: code
tags: [python]
---

# WHY

Excel 保存为 CSV 文件时，文件末尾会有很多逗号分隔的空数据行。

上传 CSV 文件时，会导致 python 后端解析报错。

# HOW

校验 CSV 数据行转为 dict 的 value 值是否 **全部** 为空：

``` python
if any(x != '' for x in d.itervalues()):
    pass
```

`itervalues()` 方法直接遍历字典，无需像 `values()` 方法再额外生成列表

**注意：** python 3 已经移除 `itervalues()` 方法：<https://www.python.org/dev/peps/pep-0469/>

python3 字典的 `d.key()`、`d.values()`、`d.items()` 方法返回的是 **迭代器** 对象，而不再是 **列表**

python2 | python3
:------ | :------
`d.iterkeys()` | `d.keys()`
`d.itervalues()` | `d.values()`
`d.iteritems()` | `d.items()`

示例代码：

``` python
with open(UPLOAD_CSV,'rb') as csv_file_input:

    dict_reader = csv.DictReader(csv_file_input) # comma is default delimiter

    # NOTE: skip empty data line only commas in CSV: ,,,,,,,,,,
    # {'oob': '', 'ip': '', 'clone': '', 'hostname': '', 'raid_array': '', 'raid_pds': '', 'role': '', 'sn': '', 'model': '', 'os': '', 'pxe_mac': ''}
    # for d in dict_reader:
    #     if any(x != '' for x in d.itervalues()):
    #         print d

    to_db = [[d[f].strip() for f in dict_reader.fieldnames] for d in dict_reader if any(v != '' for v in d.itervalues())]
```

# reference

[How to ignore blank rows in a csv file](https://stackoverflow.com/questions/8422250/how-to-ignore-blank-rows-in-a-csv-file)

[What is the difference between dict.items() and dict.iteritems() ?](https://stackoverflow.com/questions/10458437/what-is-the-difference-between-dict-items-and-dict-iteritems)

<br/>

本文标题 | [{{ page.title }}]({{ page.url }})
-------- |:--------
原始链接 | <{{ site.url }}{{ page.url }}>
