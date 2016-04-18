---
date: 2015-01-09T20:31:28+08:00
title: Django自定义User类
tags: ["django"]
---

### 参考

> [Substituting a custom User model](https://docs.djangoproject.com/en/1.5/ref/contrib/auth/#django.contrib.auth.models.User)  
> [django（v1.5） 自定义用户基础](http://django-china.cn/topic/133/)

### 实现

[Django forum](https://github.com/zhu327/forum)，学习Django了，准备写一个论坛，想法是实现一个如 <http://f2e.im/> 这样的现代论坛，数据驱动开发，第一步就是设计数据库表结构，Django自带了用户模块，但是定义的字段太少，所以需要自定义扩展下。

在`settings.py`下新增自定义用户类:
<!--more-->

```python
AUTH_USER_MODEL = 'myapp.MyUser'
```

然后在app目录的`models.py`下新增:

```python
from django.contrib.auth.models import AbstractBaseUser, AbstractUser

def MyUser(AbstractUser):
    nickname = models.CharField(max_length=30)
    ...
```

`AbstractUser`类与from django.contrib.auth.models.User的属性是一致的，`username, first_name, last_name, email, password, is_staff, is_active, is_superuser, last_login date_joined`, 所以除了已有的这些属性其余的就自由扩展了。

如果`AbstractUser`还不能满足的话，就自己派生`AbstractBaseUser`这个基础类，这个类需要写的更多，要实现一些方法，还有表单类需要重写，这些就不展开了，参考来写即可。
