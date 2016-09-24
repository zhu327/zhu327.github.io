---
date: 2016-06-14T14:16:00+08:00
title: Tornado代码阅读笔记 IOLoop
tags: ["tornado"]
---

准备用Tornado + greenlet + Django ORM搭一个框架，大体上有个思路，在开始前再次阅读下Tornado的代码。目的是在学习Torndao使用的同时，了解下原理，以便在使用过程中少踩点坑。

### 准备

1. 学习IO多路复用: epoll  
  <http://scotdoyle.com/python-epoll-howto.html>  
  <https://fukun.org/archives/10051470.html>

2. Reactor模型  
  <http://blog.csdn.net/u013074465/article/details/46276967>  

去年大概这个时候也硬着头皮读过Tornado的代码，当时没有经验，还不知道编程模型，在程序的理解上都是顺序执行的思路，所以看到Tornado的代码只会觉得特么的牛B，各种类，各种方法的调来调去。现在理解了Reactor模型以后，再回过头看Tornado，就比较容易理解了，所以在阅读代码前如果能在构架上先理解，读起来会快很多。

Tornado的代码基于2.0版本，最新的4.3版本读起来比较绕，抽象的更加细化，阅读代码的目的在于学习编程思路，不求新。

<!--more-->
### Hello World

```python
#!/usr/bin/env python

import tornado.httpserver
import tornado.ioloop
import tornado.options
import tornado.web

from tornado.options import define, options

define("port", default=8888, help="run on the given port", type=int)

class MainHandler(tornado.web.RequestHandler):
    def get(self):
        self.write("Hello, world")

def main():
    tornado.options.parse_command_line()
    application = tornado.web.Application([
        (r"/", MainHandler),
    ])
    http_server = tornado.httpserver.HTTPServer(application)
    http_server.listen(options.port)
    tornado.ioloop.IOLoop.instance().start()

if __name__ == "__main__":
    main()
```

重点看`main`函数，创建了一个HTTPServer的对象，然后就启动IOLoop了。基于对Reactor模型的理解，阅读的过程中，先不看`HTTPServer`，直接看IOLoop。

### IOLoop

Tornado中IOLoop类实现的功能对应于Reactor模型中的 反应堆， IO多路复用即epoll， 定时器集合

> <https://github.com/tornadoweb/tornado/blob/branch2.0/tornado/ioloop.py>

#### instacnce方法

Tornado是单进程单线程的，所以在同一个进程中只能有一个运行中的IOLoop，所以要实例化IOLoop是通过instance类方法实现了单例模式，在实例化过程中创建了一些实例变量，比较重要的有这些:

1. _handlers  
  保存add_handler方法添加进来的socket描述符与回调的event handler的字典
2. _callbacks  
  保存add_callback方法添加的待回调的函数
3. _timeouts  
  保存add_timeout方法添加的定时任务
4. _events  
  epoll_wait 执行后的待处理事件字典，key是scoket描述符

#### 反应堆

对应于Reactor模型中的反应堆模块，IOLoop实现了一下方法用来注册/删除/更新event handler

```python
    def add_handler(self, fd, handler, events):
        """Registers the given handler to receive the given events for fd."""
        self._handlers[fd] = stack_context.wrap(handler) # 加入到回调handler字典中
        self._impl.register(fd, events | self.ERROR) # 注册epoll 事件监听
```
添加监听的socket描述符，事件以及事件handler到_handlers字典中，并且注册监听事件到epoll上

```python
    def update_handler(self, fd, events):
        """Changes the events we listen for fd."""
        self._impl.modify(fd, events | self.ERROR)
```
更新socket描述符监听的事件

```python
    def remove_handler(self, fd):
        """Stop listening for events on fd."""
        self._handlers.pop(fd, None)
        self._events.pop(fd, None)
        try:
            self._impl.unregister(fd)
        except (OSError, IOError):
            logging.debug("Error deleting fd from IOLoop", exc_info=True)
```
移除监听的socket描述符

#### IO多路复用

start方法开启了一个无限循环，每一次循环都会做3件事:

1. 遍历_callbacks中的回调函数
2. 遍历_timeouts到时间需要处理的定时任务
3. 调用epoll_wait获取需要处理的socket事件，调用_handlers中对应socket的事件回调handler

在上面的3步中，真正的IO多路复用只是在第三步，依赖于epoll高效的事件通知机制。

#### 定时器集合

```python
    def add_timeout(self, deadline, callback):
        """Calls the given callback at the time deadline from the I/O loop.

        Returns a handle that may be passed to remove_timeout to cancel.
        """
        timeout = _Timeout(deadline, stack_context.wrap(callback))
        heapq.heappush(self._timeouts, timeout)
        return timeout

    def remove_timeout(self, timeout):
        """Cancels a pending timeout.

        The argument is a handle as returned by add_timeout.
        """
        # Removing from a heap is complicated, so just leave the defunct
        # timeout object in the queue (see discussion in 
        # http://docs.python.org/library/heapq.html).
        # If this turns out to be a problem, we could add a garbage
        # collection pass whenever there are too many dead timeouts.
        timeout.callback = None
```

以上2个方法分别是添加定时任务，以及移除定时任务。这里要注意，通过一个_Timeout类的包装，然后通过堆加入定时任务，为的是在start方法中遍历定时任务时优先处理最早的已经到时间的定时任务，在start方法里面是通过heapq。heappop方法来输出最小时间的定时任务。

#### 回调函数列表

除了Reactor模型中的反应堆，IO多路复用，定时器集合，IOLoop还实现了一个回调函数的处理机制。

```python
    def add_callback(self, callback):
        """Calls the given callback on the next I/O loop iteration.

        It is safe to call this method from any thread at any time.
        Note that this is the *only* method in IOLoop that makes this
        guarantee; all other interaction with the IOLoop must be done
        from that IOLoop's thread.  add_callback() may be used to transfer
        control from other threads to the IOLoop's thread.
        """
        if not self._callbacks:
            self._wake()
        self._callbacks.append(stack_context.wrap(callback))
```

基本上了解了这些方法，以及IO多路复用的过程，然后在看一下IOLoop docstring中的例子，整个Reactor模型中调度过程就完全实现了，然后再来看一下HTTPServer类，看看Reactor模型中concrete event handler的实现。

### HTTPServer

在HTTPServer实例化过程中保存了处理请求的Application对象。调用listen方法，listen方法调用了2个方法:

1. bind
  创建了一个非阻塞的socket，并bind，listen  
2. start

```python
            if not self.io_loop:
                self.io_loop = ioloop.IOLoop.instance() # 创建 ioloop 对象
            for fd in self._sockets.keys(): # 把socket字典中所有的socket加入到ioloop handler列表中，并监听可读事件，回调函数为_handle_events
                self.io_loop.add_handler(fd, self._handle_events,
                                         ioloop.IOLoop.READ)
```
只看这一段，创建了IOLoop实例，调用add_handler注册soket监听可读事件，event handler回调函数是_handle_events方法。

#### _handle_events

```python
    def _handle_events(self, fd, events): # 当接受到服务端socket的可读事件时，建立连接
        while True:
            try:
                connection, address = self._sockets[fd].accept() # 接受socket连接
            except socket.error, e:
                if e.args[0] in (errno.EWOULDBLOCK, errno.EAGAIN):
                    return
                raise
            
            stream = iostream.IOStream(connection, io_loop=self.io_loop) # 创建IOStream对象，封装子socket
            HTTPConnection(stream, address, self.request_callback,
                               self.no_keep_alive, self.xheaders) # 创建 HTTPonnection连接对象，处理http请求
```
省略了大部分，只看关键的地方。_handle_events方法实际上就是Reactor模型概率中的concrete event handler。

在epoll_wait的过程中，如果主socket有可读事件则会回调_handle_events方法，调用accept方法接受子socket，子socket封装为IOStream对象，封装了socket读写相关的操作，使用IOStream对象实例化HTTPConnection对象就完成了整个concrete event handler的过程。

### 总结

以上就是我在阅读IOLoop代码的一些理解，接下来将重点阅读IOStream相关的代码，以及Tornado协程相关的代码。


