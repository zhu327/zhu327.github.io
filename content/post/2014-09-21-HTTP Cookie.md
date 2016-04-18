---
date: 2014-09-21T12:39:04+08:00
title: HTTP Cookie
tags: ["http", "web"]
---

### 什么是Cookie

Cookie用于存储http请求中的用户认证信息，用户在通过登录认证后，服务器在response header中设置Cookie，用户浏览器自动带Cookie访问该网站。

### response 中的 Cookie

response header 中添加 `Set-Cookie`信息，cookie是一组key，value值。  

    Set-Cookie: <name>=<value>[; <name>=<value>]...
                [; expires=<date>][; domain=<domain_name>]
                [; path=<some_path>][; secure][; httponly]

<!--more-->
* expires Cookie过期时间
* domain Cookie的生效域
* path Cookie的生效路径
* secure 表示cookie只能被发送到http服务器
* httponly 表示cookie不能被客户端脚本获取到

不带expires的Cookie表示临时Cookie只在当前会话有效，关闭浏览器Cookie实效。

### request 中的 Cookie

浏览器在接受的response中如果发现有`Set-Cookie`头，则在下一个request中设置`Cookie`头，如果有expires还要判断Cookie是否过期，reqeust中的cookie没有expires字段，
如果是临时Cookie，浏览器直接保存在内存中，带expire的cookie如果未过期保存硬盘中，浏览器自动删除过期的Cookie，所有服务器要删除Cookie就是重新发送带`Set-Cookie`并且expires
早与当前时间的Cookie。

### 关于 session

session是保存在服务端的会话控制，一般服务器会发送包含sessionid的临时Cookie给浏览器，服务器根据sessionid获取保存的用户信息，自行维护过期时间等信息。
