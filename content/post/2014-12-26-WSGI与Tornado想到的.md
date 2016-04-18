---
date: 2014-12-26T23:59:11+08:00
title: WSGI与Tornado想到的
tags: ["wsgi", "tornado", "python", "web"]
---

### Tornado

上周用tornado实现了[RSS Factory](https://github.com/zhu327/rss)，又从新学了下Tornado下的使用，再次看[introduce to tornado](http://demo.pythoner.com/itt2zh/index.html)发现以前看的只学到了Tornado的MVC，使用Tornado也只停留在表面，没有学到Tornado的精髓。这次又学到了不少，这里总结下。  

### 长连接,WebSocket与异步事件循环

Web2.0时代产生了很多的实时通知需求，也就是Ajax请求特别多，最初是用轮询实现，但是对服务器的开销很大，所以产生了长连接轮询。

客户的也就是浏览器请求一个api，Tornado不会立即返回，而是等到有变化的时候返回，浏览器获取响应后，循环再请求api，这样就省掉了很多请求，而且实现了实时通知。

<!--more-->
再说WebSocket，浏览器与服务器都是在网页中建立一次连接，后续的数据双向通信都是基于事件通知的，对应事件产生不同的消息而不需要重复请求。

这里就可以看到Tornado最初设计的使用场景就是做实时通知，服务器的核心就是基于epoll的异步事件循环，客户的请求发送到事件循环处理，可以异步处理，而不占用资源。RSS Factory就是用到了Tornado的异步功能才在同时fetch多个url的情况下，速度也不会太慢。服务器的并发性能非常高。

但是同时Tornado也不是万能的，因为不同于WSGI实现通过线程实现并发，Tornado并没有线程的概念，只是使用事件循环来实现并发，这样如果有一个请求的处理对象中有同步阻塞，那么所有的请求都会被阻塞，所以写Tornado app就变成了不得不异步。

### WSGI

最早开始学Python的时候就是为了实现微博RSS，那时候刚开始学Django，大部分网络上的部署方案还是nginx+uWSGI+Django，wsgi的并发方案一般都是线程实现的，每个请求对应一个线程，这样每个请求都能独立处理。但是对系统资源占用比较大，所以各个WSGI服务器的实现性能不同。

### gunicorn与gevent

以前部署wsgi应用都是用uWSGI，一个C实现的WSGI服务器，性能好于fastCGI，现在主流都是使用gunicorn来做wsgi服务器，gunicorn并发性能也挺好，但是搭配gevent性能会更好，因为gevent抛弃了线程来并发，而是用协程(相当于轻量级线程)来替代线程，所以并发性能基本上是Python服务器中最好的，而且使用gevent不需要修改代码即可直接使用，所以推荐使用nginx+gunicorn(gevent)来部署wsgi应用。
