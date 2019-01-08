---
layout: post
title: "手动自定义排列 Django Admin 主页 APP 列表顺序"
category: code
tags: [python, django]
---

# WHY

Django Admin 主页 APP 列表默认是按照 **字母顺序** 进行排列的，想根据 APP 使用来排序。

<https://docs.djangoproject.com/en/2.1/intro/tutorial07/>

> By default, it displays all the apps in `INSTALLED_APPS` that have been registered with the admin application, **in alphabetical order**.

<https://docs.djangoproject.com/en/2.1/_modules/django/contrib/admin/sites/>

``` python
def get_app_list(self, request):
    """
    Return a sorted list of all the installed apps that have been
    registered in this site.
    """
    app_dict = self._build_app_dict(request)

    # Sort the apps alphabetically.
    app_list = sorted(app_dict.values(), key=lambda x: x['name'].lower())

    # Sort the models alphabetically within each app.
    for app in app_list:
        app['models'].sort(key=lambda x: x['name'])

    return app_list
```

网上有多种实现方案：

- 在 **每个** APP model 里为 Meta 类下的 `verbose_name_plural` 字段添加 **数字前缀** <sup>[1](https://stackoverflow.com/questions/21578382/reorder-model-objects-in-django-admin-panel)</sup>
- 重写 `admin/base.html` 模板页面 <sup>[2](https://jamiecounsell.me/sort-apps-in-the-django-admin/)</sup>
- 重写 `AdminSite` 类的 `index()` 方法 <sup>[3](https://stackoverflow.com/questions/8893755/defining-a-custom-app-list-in-django-admin-index-page/)</sup>
- 重新 `AdminSite` 类的 `get_app_list()` 方法 <sup>[4](https://books.agiliq.com/projects/django-admin-cookbook/en/latest/set_ordering.html)</sup>

本文使用第三种：重写 `AdminSite` 类的 `index()` 方法

# HOW

Django 源码 Admin 主页渲染的 `app_list` 列表是由 `get_app_list()` 方法按 **字母排序** 后返回的结果

所以只需要对 `app_list` 进行排序即可

<https://docs.djangoproject.com/en/2.1/_modules/django/contrib/admin/sites/>

``` python
def index(self, request, extra_context=None):
    """
    Display the main admin index page, which lists all of the installed
    apps that have been registered in this site.
    """
    app_list = self.get_app_list(request)

    context = {
        **self.each_context(request),
        'title': self.index_title,
        'app_list': app_list,
        **(extra_context or {}),
    }

    request.current_app = self.name

    return TemplateResponse(request, self.index_template or 'admin/index.html', context)
```

首先参考官方文档覆盖 Django Admin 的 `AdminSite` 类：

<https://docs.djangoproject.com/en/2.1/ref/contrib/admin/#overriding-the-default-admin-site>

项目名称 `zerone`

在 `zerone/` 子目录下创建 `zerone/admin.py` 文件，重写 `AdminSite` 的 `index()` 方法：

``` python
from django.contrib import admin

class MyAdminSite(admin.AdminSite):

    def index(self, request, extra_context=None):
        if extra_context is None:
            extra_context = {}

        app_list = self.get_app_list(request)
        app_dict = self._build_app_dict(request)

        # dict_keys(['equipment', 'liveos', 'clone', 'oauth2_provider', 'auth'])
        print(app_list, app_dict.keys())

        extra_context['app_list'] = app_list
        return super(MyAdminSite, self).index(request, extra_context)
```

创建 `zerone/apps.py` 文件，自定义 `AdminConfig` 配置名称：

``` python
from django.contrib.admin.apps import AdminConfig

class MyAdminConfig(AdminConfig):
    default_site = 'zerone.admin.MyAdminSite'
```

使用刚在 `zerone/apps.py` 自定义的 APP 配置名称，替换默认的 `django.contrib.admin`

修改 `zerone/settings.py` ：

    INSTALLED_APPS = [
        ...
        'zerone.apps.MyAdminConfig',  # replaces 'django.contrib.admin'
        ...
    ]

重写 `AdminSite` 的 `index()` 方法之前，先查看一下默认 `app_dict` 包含的 **所有** APP 名称：

``` python
app_list = self.get_app_list(request)
app_dict = self._build_app_dict(request)
print(app_list, app_dict.keys())

# dict_keys(['equipment', 'liveos', 'clone', 'oauth2_provider', 'auth'])
```

默认 `get_app_list()` 按照字母排序的 APP 顺序是：

    'auth',
    'oauth2_provider',
    'clone',
    'equipment',
    'liveos'

然后根据上面显示的 **所有** APP 名称，自定义 APP 的排序列表：

    apps_order_list = ['equipment', 'liveos', 'clone', 'oauth2_provider', 'auth']

然后就是根据自定义的 `apps_order_list` 对默认的 `app_list` 进行排序：

``` python
from django.contrib import admin

apps_order_list = ['equipment', 'liveos', 'clone', 'oauth2_provider', 'auth']
apps_order_dict = { app: index for index, app in enumerate(apps_order_list) }

class MyAdminSite(admin.AdminSite):

    def index(self, request, extra_context=None):
        if extra_context is None:
            extra_context = {}

        app_list = self.get_app_list(request)
        app_dict = self._build_app_dict(request)

        # dict_keys(['equipment', 'liveos', 'clone', 'oauth2_provider', 'auth'])
        print(app_dict.keys())

        # https://stackoverflow.com/questions/15650348/sorting-a-list-of-dictionaries-based-on-the-order-of-values-of-another-list
        # https://docs.python.org/3/howto/sorting.html
        app_list.sort(key=lambda element_dict: apps_order_dict[element_dict["app_label"]])

        # print(app_list)

        extra_context['app_list'] = app_list
        return super(MyAdminSite, self).index(request, extra_context)
```

DONE

# reference

<https://docs.python.org/3/howto/sorting.html>

> You can also use the `list.sort()` method. It **modifies the list in-place** ( and returns `None` to avoid confusion ).
> Usually it’s less convenient than `sorted()` - but if you don’t need the original list, it’s slightly **more efficient**.

[Sorting a list of dictionaries based on the order of values of another list](https://stackoverflow.com/questions/15650348/sorting-a-list-of-dictionaries-based-on-the-order-of-values-of-another-list)

[Defining a custom app_list in django admin index page](https://stackoverflow.com/questions/8893755/defining-a-custom-app-list-in-django-admin-index-page/)

[Reorder model objects in django admin panel](https://stackoverflow.com/questions/21578382/reorder-model-objects-in-django-admin-panel)

[Ordering admin.ModelAdmin objects in Django Admin](https://stackoverflow.com/questions/398163/ordering-admin-modeladmin-objects-in-django-admin/)

[How to set ordering of Apps and models in Django admin dashboard.](https://books.agiliq.com/projects/django-admin-cookbook/en/latest/set_ordering.html)

<br/>

本文标题 | [{{ page.title }}]({{ page.url }})
-------- |:--------
原始链接 | <{{ site.url }}{{ page.url }}>
