---
date: 2016-05-15T10:49:01+08:00
title: Tastypie实现accesstoken认证
tags: ["django", "tastypie"]
---

Tastypie提供了几种基本的认证方式比如SessionAuthentication，Django实现的web站点一般都是基于cookie-session的认证方式，在Django中使用中间件的方式处理cookie与session，以及用户认证。使用起来是很方便的。

但是在api中就不能继续使用cookie了，我们会使用accesstoken的方式来实现认证。之前同事使用redis实现了一版accesstoken的认证，但是由于考虑不周全，导致在使用过程中出现了很多bug，然后为了解决bug又写了很多恶心的兼容代码，所以抽空直接使用熟悉的session重构了认证过程。

<!--more-->

```python
# coding: utf-8

from django.conf import settings
from django.contrib.sessions.middleware import SessionMiddleware
from django.utils.functional import SimpleLazyObject
from tastypie.authentication import Authentication

from apps.ucenter.models import UserInfo


SESSION_KEY = '_auth_user_id'


class EhrSessionMiddleware(SessionMiddleware):
    '''
    定制session中间件，继承至Django自己的SessionMiddleware
    '''

    def process_request(self, request):
        session_key = request.META.get('HTTP_ACCESSTOKEN') # 直接在http头里取accesstoken，作为session的key
        request.session = self.SessionStore(session_key) # 获取到request session

    def process_response(self, request, response):
        """
        If request.session was modified, or if the configuration is to save the
        session every time, save the changes and set a session cookie or delete
        the session cookie if the session has been emptied.
        这里就直接抄来，去掉了set_cookie相关的代码
        """
        try:
            modified = request.session.modified
        except AttributeError:
            pass
        else:
            if modified or settings.SESSION_SAVE_EVERY_REQUEST:
                if response.status_code != 500:
                    request.session.save()
        return response


def get_user(request):
    '''
    获取用户信息
    '''
    if not hasattr(request, '_cached_user'):
        user_id = request.session.get(SESSION_KEY)
        user = user_id and UserInfo.objects.filter(pk=user_id).first() or None
        request._cached_user = user
    return request._cached_user


class AuthenticationMiddleware(object):
    def process_request(self, request):
        assert hasattr(request, 'session'), (
            "The Django authentication middleware requires session middleware "
            "to be installed. Edit your MIDDLEWARE_CLASSES setting to insert "
            "'django.contrib.sessions.middleware.SessionMiddleware' before "
            "'django.contrib.auth.middleware.AuthenticationMiddleware'."
        )
        '''
        认证中间件，也是抄django.contrib.auth.middleware.AuthenticationMiddleware
        '''
        request.user = SimpleLazyObject(lambda: get_user(request))


def login(request, user):
    """
    生成token列表
    """
    if user is None:
        user = request.user

    request.session[SESSION_KEY] = str(user.pk)
    request.session.save()
	return {'accesstoken': request.session.session_key} # 使用session作为token


def logout(request):
    '''
    退出
    '''
    request.session.flush()
    request.user = None


class UserAuthentication(Authentication):
    '''
    认证是否登录，在Tastypie中使用，用于判断request是否已认证
    '''
    def is_authenticated(self, request, **kwargs):
        return isinstance(request.user, UserInfo)
```

这样使用我们熟悉的session逻辑就实现了我们需要的accesstoken认证，只需要在settings中的中间件配置中注释掉原来的SessionMiddleware，AuthenticationMiddleware中间配置上上面的版本，然后在登录，退出逻辑中使用login，logout函数即可。

```python
MIDDLEWARE_CLASSES = [
    # 'django.contrib.sessions.middleware.SessionMiddleware', 注释掉原来的session中间件
    'core.authentication.EhrSessionMiddleware',
    'django.middleware.common.CommonMiddleware',
    # 'django.middleware.csrf.CsrfViewMiddleware',
    # 'django.contrib.auth.middleware.AuthenticationMiddleware', 注释掉原来的认证中间件
    'core.authentication.AuthenticationMiddleware',
    # 'django.contrib.messages.middleware.MessageMiddleware',
    'django.middleware.clickjacking.XFrameOptionsMiddleware',
    'django.middleware.locale.LocaleMiddleware',
    'core.middleware.XForwardedForMiddleware',
]
```