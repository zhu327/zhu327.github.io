---
date: 2015-01-31T11:23:21+08:00
title: Cookie与Session
tags: ["web"]
---

前面学习了Cookie，知道Cookie是在浏览器端保存的用户状态，但是对Session一直没什么概念。学习Django的过程中直接使用了Django的认证系统，虽然用到了Session但是没有接触到Session的直接使用，然后在学习F2E.im的代码中，发现Tornado自带的secret cookie其实也能加密传递cookie，通过cookie传递一个userid到用户浏览器，认证的时候使用userid到数据库中查找用户信息，也能很好的认证用户。那为什么要使用session呢。

Tonado中认证示例:

```python
# 重写tornado.web.RequestHandler中get_current_user方法用于认证
def get_current_user(self):
    user_id = self.get_secure_cookie("user")
    if not user_id: return None
    return self.user_model.get_user_by_uid(int(user_id)) # 从数据库中取用户信息
```
<!--more-->

带着疑惑询问了某同事，得到了一个比较靠谱的答案。

如上面的示例代码，在验证了用户名密码后，登录就把Cookie发送给用户:

```python
self.set_secure_cookie("user", str(user_id)) # Tornado中设置Cookie
```

logout时清除Cookie:

```python
self.clear_cookie("user")
```

但是这里有个问题即使这个Cookie是已经加密过的，但是只要加密密钥不变，这个Cookie如果被截取下来，clear_cookie只会清空当前浏览器的Cookie，不保证Cookie被有心的保存下来，如果继续使用被保存下来的Cookie，服务器是不会知道这个在有效时间内的Cookie是不是真的被退出。

那么问题来了，怎么样保证用户登出后，Cookie实效呢，答案是用随机Cookie，使用Session，通过session保存一个随机Cookie作为key，value为用户id或者其它什么用户信息，用户登录的时候生成随机字符串，一般用sessionid作为Cookie的key，随机字符串作为value发送给用户，在Session中用这个随机字符串作为key，vaule为用户信息，在这个session有效时，首先是到session中获取用户信息。比如这样:

```python
def get_current_user(self):
    sessionid = self.get_secure_cookie("sessionid")
    if not sessionid: return None
    user_id = seesion.get(sessionid, None) # 先从seesion中获取用户信息
    if not user_id: return None
    return self.user_model.get_user_by_uid(int(user_id)) # 从数据库中取用户信息
```

登录:

```python
sessionid = session.id() # 假设有这么一个生成随机字符串的方法
session.set(sessionid, userid)
self.set_secure_cookie("sessionid", str(sessionid))
```

登出:

```python
session.delete(sessionid)
self.clear_cookie("sessionid")
```

这样即使sessionid被盗取，只要用户登出后，这个sessionid就失效了，就不会有Cookie在有效期内一直有效的问题了。

从以上示例就可以看出session实现了什么功能:

1. session是在服务端保存用户状态的东西
2. session是 key-value存储信息的
3. session有一个实现随机字符串的方法
