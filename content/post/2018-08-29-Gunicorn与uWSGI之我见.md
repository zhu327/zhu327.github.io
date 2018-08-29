---
title: "Gunicorn与uWSGI之我见"
date: 2018-08-29T23:23:20+08:00
draft: false
---

昨天前同事问我[doge](https://github.com/zhu327/doge)的服务端怎么是单进程跑的, 其实在生产环境下我们参考[gunicorn_thrift](https://github.com/eleme/gunicorn_thrift)实现了一个定制的master/worker模型的Gunicorn服务器. 昨天也写了一个[简化版本](https://github.com/zhu327/doge/tree/master/doge/gunicorn)集成到doge, 实际代码不超过20行就能利用到Gunicorn的进程管理功能. 有感于Gunicorn简洁优雅的模型, 这里聊聊我理解的Gunicorn与uWSGI.

## perfork

perfork是一种服务端编程模型, Nginx, Gunicorn, uWSGI都是这种模型的实现, 简单的说perfok就是master进程启动注册一堆信号处理函数, 创建listen socket fd, fork出多个worker子进程, 子进程执行accept循环处理请求(这里简化模型, 当然也可以用select, epoll多路复用), master进程只负责监控worker进程状态, 通过pipeline通信来控制worker进程.

<!--more-->

perfork模型使用master进程来监控worker进程状态, 避免了我们使用supervisor来监控进程, 还支持多种信号来控制worker的数量, 使得CPU能充分得到利用, 多个worker进程监听同一端口, 可以配置reuse_port参数在worker进程间负载均衡.

Gunicorn与uWSGI都是基于perfork模型的WSGI服务器, 下面我们就来聊聊他们的区别.

## Gunicorn

Gunicorn是使用Python实现的WSGI服务器, 直接提供了http服务, 并且在woker上提供了多种选择, gevent, eventlet这些都支持, 在多worker最大化里用CPU的同时, 还可以使用协程来提供并发支撑, 对于网络IO密集的服务比较有利.

同时Gunicorn也很容易就改造成一个TCP的服务, 比如[doge](https://github.com/zhu327/doge/tree/master/doge/gunicorn)重写worker类, 在针对长连接的服务时, 最好开启reuse_port, 避免worker进程负载不均.

## uWSGI

不同于Gunicorn, uWSGI是使用C写的, 它的socket fd创建, worker进程的启动都是使用C语言系统接口来实现的, 在worker进程处理循环中, 解析了http请求后, 使用python的C接口生成environ对象, 再把这个对象作为参数塞到暴露出来的WSGI application函数中调用. 而这一切都是在C程序中进行, 只是在处理请求的时候交给python虚拟机调用application. 完全使用C语言实现的好处是性能会好一些.

除了支持http协议, uWSGI还实现了uwsgi协议, 一般我们会在uWSGI服务器前面使用Nginx作为负载均衡, 如果使用http协议, 请求在转发到uWSGI前已经在Nginx这里解析了一遍, 转发到uWSGI又会重新解析一遍. uWSGI为了追求性能, 设计了uwsgi协议, 在Nginx解析完以后直接把解析好的结果通过uwsgi协议转发到uWSGI服务器, uWSGI拿到请求按格式生成environ对象, 不需要重复解析请求. 如果用Nginx配合uWSGI, 最好使用uwsgi协议来转发请求.

除了是一个WSGI服务器, uWSGI还是一个开发框架, 它提供了缓存, 队列, rpc等等功能, 在github找找就会发现有人用它的缓存写了一个Django cache backend, 用它的队列实现异步任务这些东西, 但是用了这些东西技术栈也就跟uWSGI绑定在一起, 所以一般也只是把uWSGI当作WSGI服务器来用.

## Nginx

使用多个进程监听同一端口就绕不开惊群这个话题, fork子进程, 子进程共享listen socket fd, 多个子进程同时accept阻塞, 在请求到达时内核会唤醒所有accept的进程, 然而只有一个进程能accept成功, 其它进程accept失败再次阻塞, 影响系统性能, 这就是惊群. Linux 2.6内核更新以后多个进程accept只有一个进程会被唤醒, 但是如果使用epoll还是会产生惊群现象.

Nginx为了解决epoll惊群问题, 使用进程间互斥锁, 只有拿到锁的进程才能把listen fd加入到epoll中, 在accept完成后再释放锁. 

但是在高并发情况下, 获取锁的开销也会影响性能, 一般会建议把锁配置关掉. 直到Nginx 1.9.1更新支持了socket的`SO_REUSEPORT`选项, 惊群问题才算解决, listen socket fd不再是在master进程中创建, 而是每个worker进程创建一个通过`SO_REUSEPORT` 选项来复用端口, 内核会自行选择一个fd来唤醒, 并且有负载均衡算法.

Gunicorn与uWSGI都支持reuse_port选项, 在使用时可以通过压测来评估一下reuse_port是否能提升性能.

一般我们会在Gunicorn/uWSGI前面再加一层Nginx, 这样做的原因有一下几点:

1. 做负载均衡
2. 静态文件服务器
3. 更安全
4. 扛并发

## 参考

<https://github.com/Junnplus/blog/issues/9>

<https://gunicorn.readthedocs.io/en/latest/index.html>

<https://www.nginx.com/blog/socket-sharding-nginx-release-1-9-1/>

<https://www.zhihu.com/question/38528616>