---
date: 2015-03-16T23:12:18+08:00
title: Django实现简单OAuth2.0认证服务
tags: ["web", "django"]
---

开始写[Django forum](https://github.com/zhu327/forum)的RESTful api，首先解决用户认证的问题，使用OAuth2.0协议实现。

### 参考

[理解OAuth 2.0](http://www.ruanyifeng.com/blog/2014/05/oauth_2_0.html)

### 授权

OAuth2.0协议定义了4种授权模式，为了学习OAuth2.0授权协议，这里只实现简化模式。以下为简化模式授权过程。

<!--more-->
1. 客户端对认证URI`/api/authorize`发起`GET`请求，必须带参数：
   * `response_type`：表示授权类型，此处的值固定为"token"，必选项。
   * `client_id`：表示客户端的ID，必选项。
   * `redirect_uri`：表示重定向的URI，可选项。
   * `state`：表示客户端的当前状态，可以指定任意值，认证服务器会原封不动地返回这个值。
2. 服务器验证参数返回登陆页面；
3. 用户输入用户名，密码提交登陆`POST`表单;
4. 服务器验证表单，生成access_token，并重定向到`redirect_uri`，附带参数：
   * `access_token`：表示访问令牌，必选项。
   * `token_type`：表示令牌类型，该值大小写不敏感，必选项。
   * `expires_in`：表示过期时间，单位为秒。如果省略该参数，必须其他方式设置过期时间。
   * `state`：如果客户端的请求中包含这个参数，认证服务器的回应也必须一模一样包含这个参数。

然后客户端就可以使用`access_token`来请求Django forum实现的需要认证的RESTful api了。

### 实现

了解了认证过程后，实现起来就比较简单了，为了学习Django class based view，这里尝试不用function based view实现。

> <https://github.com/zhu327/forum/blob/api/api/views/oauth.py>

```python
# coding: utf-8

'''
Oauth2.0 认证
'''

import json, hashlib, time, base64, urllib
from django.shortcuts import render_to_response, redirect
from django.http import HttpResponse
from django.views.generic import View
from django.template import RequestContext
from django.conf import settings

from forum.models import ForumUser
from forum.forms.user import LoginForm
from api.forms.oauth import OauthForm


_OAUTH_ERROR = {
    '0001': 'unkown_client_id',
    '0002': 'redirect_uri_mismatch',
    '0003': 'unsupported_response_type',
    '0004': 'expired_token',
    '0005': 'login_failed',
    '0006': 'invalid_access_token'
}


def _oauth_error(code):
    error = {
        'error': code,
        'error_description': _OAUTH_ERROR.get(str(code), 'unkown_error')
    }
    return HttpResponse(json.dumps(error), content_type='application/json')


# 生成access_token
def make_access_token(client_id, id, password, max_age=5184000000):
    expires = str(int(time.time()) + max_age)
    L = [str(id), expires, hashlib.md5('%s-%s-%s-%s-%s' % (client_id, id,\
        password, expires, settings.SECRET_KEY)).hexdigest()]
    return base64.encodestring('-'.join(L)), expires


# 解析access_token
def parse_access_token(client_id, token):
    try:
        L = base64.decodestring(token).split('-')
        if len(L) != 3:
            return _oauth_error('0006')
        id, expires, md5 = L
        if int(expires) < time.time():
            return _oauth_error('0004')
        try:
            user = ForumUser.get(pk=id)
        except ForumUser.DoesNotExist:
            return _oauth_error('0006')
        if md5 != hashlib.md5('%s-%s-%s-%s-%s' % (client_id, id, user.password, expires, settings.SECRET_KEY)).hexdigest():
            return _oauth_error('0006')
        return user
    except:
        return _oauth_error('0006')


# 装饰器,用于认证access_token,类似于Django自带的login_required使用
def login_required(func):
    def _wrapped_view(request, *args, **kwargs):
        client_id = request.REQUEST.get('client_id', None)
        access_token = request.REQUEST.get('access_token', None)
        if client_id and access_token:
            r = parse_access_token(client_id, access_token)
            if isinstance(r, HttpResponse):
                return r
            request.user = r
            return func(request, *args, **kwargs)
        return _oauth_error('0006')
    return _wrapped_view


class OauthView(View):
    def get(self, request):
        '''
        验证QueryString并返回登录页面
        '''
        form = OauthForm(request.GET)
        if not form.is_valid():
            if form['response_type'].errors:
                return _oauth_error('0003')
            elif form['client_id'].errors:
                return _oauth_error('0001')
            elif form['redirect_uri'].errors:
                return _oauth_error('0002')
        return render_to_response('user/login.html', context_instance=RequestContext(request))

    def post(self, request):
        '''
        登录成功后返回access_token
        '''
        get_form = OauthForm(request.GET)
        if not get_form.is_valid():
            if get_form['response_type'].errors:
                return _oauth_error('0003')
            elif get_form['client_id'].errors:
                return _oauth_error('0001')
            elif get_form['redirect_uri'].errors:
                return _oauth_error('0002')

        post_form = LoginForm(request.POST)
        if not post_form.is_valid():
            return render_to_response('user/login.html', {'errors': post_form.errors},\
                context_instance=RequestContext(request))

        user = post_form.get_user()
        access_token, expires_in = make_access_token(get_form.cleaned_data.get('client_id'), user.id, user.password)
        params = {
            'access_token': access_token,
            'token_type': 'token',
            'expires_in': expires_in,
        }
        if get_form.cleaned_data.get('state', None):
            params['state'] = get_form.cleaned_data.get('state')
        return redirect('%s?%s' % (get_form.cleaned_data.get('redirect_uri'), urllib.urlencode(params)))
```

以上代码复用了很多Django forum以前实现的东西，比如登陆页面，认证表单等等，实现`make_access_token`函数用于生成`access_token`，
`parse_access_token`用于解析`access_token`，并且实现了`login_required`装饰器用来包裹需要认证的api。

### 总结

这里为了方便，只实现了OAuth2.0协议的简单模式，实际上互联网大部分的公开api认证，比如新浪微博都是使用的授权码模式，有了简单模式的经验，
实现授权码模式也很简单，区别在生成`access_token`的时候同时生成授权码`code`，以`code`为key，
`clien_id`,`access_token`等参数的字典为value，放入memcached缓存中，设置过期时间，然后重定向到`redirect_uri`并带上`code`，
第三方服务器再用`code`请求`/api/token`，服务器查找`code`为key的value，返回`access_token`给第三方服务器，结束授权。
