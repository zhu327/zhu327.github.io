---
date: 2015-01-31T13:22:55+08:00
title: Django forum总结
tags: ["Django"]
---

<https://github.com/zhu327/forum>

元旦的时候决定开始学习Django，学习最好的方式无非边学边做，所以把2年前没敢看的一个Tornado项目通过用Django实现了，这里总结下。

### models

1. 扩展默认User类

    ```python
    from django.contrib.auth.models import AbstractUser
    # 继承这个类即可

    # setting中设置
    AUTH_USER_MODEL = 'forum.ForumUser' # 指定用户对象使用的类
    ```
<!--more-->

2. 扩展Model的方法

    ```python
    # 默认Model对象有个属性objects有各种方法，扩展这个方法
    from django.db.models import Manager
    # 扩展这个类
    # self.get_query_set() 这个方法返回基础的QuerySet，所以扩展Manager都是围绕这个方法来写
    # 在Model类中把objects指向这个Manager的对象
    class Topic(models.Model):
        ...
        objects = TopicManager() # 默认objects就是model.Manager的对象
    ```

3. QuerySet对象的一些方法

    ```python
    # 复杂查询使用join连接方法，用于1对1的对象，一般ForeignKey
    QuerySet.select_related()
    # 复杂查询，用于 _set的提前查询，用于1对多，多对多，ForeignKey的被指向对象都可以，通过SQL in方法
    QuerySet.prefetch_related()
    # exits() 判断是否有结果
    QuerySet.exits()
    # bulk_create() 批量创建，参数为Model对象的列表
    Model.objects.bulk_create()
    ```

### AUTH

[Django forum](https://github.com/zhu327/forum)用到了Django自带的认证系统，但是做了很多的自定义内容。

1. settings

    ```python
    # 自定义User类
    AUTH_USER_MODEL = 'forum.ForumUser'

    # 用户认证BackEnds
    AUTHENTICATION_BACKENDS = ('forum.backends.EmailAuthBackend',)

    # 默认登陆uri
    LOGIN_URL = '/login/'
    ```

    这里重点说一下`AUTHENTICATION_BACKENDS`，这个Django的认证后端，在使用Django的`auth.login(request, user)`登录用户的时候，user对象必须是由`authenticate`返回才行，否则会抛出没有BackEnds的异常。而`authenticate`就是通过`settings.AUTHENTICATION_BACKENDS`来认证用户的，并返回带BackEnds对象的User实例。默认的`AUTHENTICATION_BACKENDS`使用用户名与密码来来认证，Django forum使用email与密码认证所以需要重写认证后端。

    ```python
    from django.contrib.auth.backends import ModelBackend # 继承这个为了使用admin的权限控制
    from forum.models import ForumUser

    class EmailAuthBackend(ModelBackend):

        def authenticate(self, username=None,   password=None):
            try:
                user = ForumUser.objects.get(email=username)
                if user.check_password(password):
                    return user
                    return None
                except ForumUser.DoesNotExist:
                    return None

            def get_user(self, user_id):
            try:
                return ForumUser.objects.get(pk=user_id)
            except ForumUser.DoesNotExist:
                return None
    ```

    这个自定义认证后端继承自`ModelBackend`，需要实现2个方法`authenticate`方法用于认证，返回None或者用户对象，`get_user`参数为用户id，返回用户对象或`None`。

2. Django admin使用自定义的登录页面

    Django admin有自带的登录页面，如何用自己的登录页面来登录admin呢。

    ```python
    from django.contrib import admin
    admin.autodiscover()
    admin.site.login = login_required(admin.site.login) # 设置admin登录的页面，settings.LOGIN_URL
    ```

    在admin所在的`urls.py`中做以上设置，访问admin即可自动重定向到`settings.LOGIN_URL`设置的url上，并且带上`next`参数在登录后返回admin。

### template

1. 自定义模板filter

    ```python
    from django import template
    register = template.Library()
    @register.filter(name='')
    def custom_filter():
        ...
    ```

    但是这个自定义多虑期只能额外有1个参数，如何用多个参数呢。用自定义tag

2. 自定义tag

    ```python
    @register.simple_tag # tag名即为函数名
    def func():
        ...
    ```

3. 一些内置filter

    ```python
    add # 用来求和
    linebreaks # 对字符串中的\n替换为</br>，\n\n替换为<p></p>
    ```

### Cache

使用memcached:

```python
# settings
CACHES = { # memcached缓存设置
    'default': {
        'BACKEND': 'django.core.cache.backends.memcached.MemcachedCache',
        'LOCATION': '127.0.0.1:11211',
    }
}

SESSION_ENGINE = 'django.contrib.sessions.backends.cache' # 使用memcached存储session
```
