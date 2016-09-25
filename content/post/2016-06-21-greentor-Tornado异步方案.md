---
date: 2016-06-21T18:51:38+08:00
title: greentor Tornado异步方案
tags: ["tornado", "greentor"]
---

> <https://emptysqua.re/blog/motor-internals-how-i-asynchronized-a-synchronous-library/>

这篇文章是Motor的作者介绍Motor如何通过Greenlet来实现PyMongo在Tornado中异步调用的原理，总结来说就一下几点。

1. 使用Torando的IOStream包装socket以实现异步调度
2. 把IOStream的读写操作放在greenlet中运行，并注册一个switch到当前greenlet的callback到IOStream的Futrue中
3. 在发生读写操作是switch到当前greenlet的父greenlet继续执行，挂起当前greenlet
4. 在IOStream的读写操作完成后调用callback switch到挂起的子greenlet中继续执行

```python
def tornado_motor_sock_method(method):
    coro = gen.coroutine(method)

    @functools.wraps(method)
    def wrapped(self, *args, **kwargs):
        #当前greenlet是一个子greenlet
        child_gr = greenlet.getcurrent()
        #获取当前greenlet的父greenlet，即之前代码提到过的asynchronize所在的greenlet
        main = child_gr.parent

        def callback(future):
            if future.exc_info():
                child_gr.throw(*future.exc_info())
            elif future.exception():
                child_gr.throw(future.exception())
            else:
                #当future的结果到达，切换回挂起的子greenlet
                child_gr.switch(future.result())

        #保证callback在当前greenlet的父greenlet中运行
        self.io_loop.add_future(coro(self, *args, **kwargs), callback)
        #return这句会暂时挂起当前greenlet，将控制权切换回父greenlet，
        #在上面的callback执行时，才会切换回当前greenlet，return语句返回
        return main.switch()
    return wrapped
```

使用greenlet的好处是我们可以通过这个挂起，唤醒的过程来中断当前的同步代码，而不需要用Tornado自己实现协程，每次都要yield出来，然后回调。通过使用greenlet可以很方便的把同步的网络IO库修改为支持Tornado的异步库。

<!--more-->
### greentor

> <https://github.com/zhu327/greentor>

基于以上原理，greentor实现了一个`AsyncSocket`异步socket类，用IOStream来包装socket，并且在读写操作的时候切换到当前greenlet的父greenlet.

然后再修改pymysql的connect过程，用`AsyncSocket`替换socket，打上补丁后就可以使用异步IO了。但是涉及到IO操作的地方都需要运行在子greenlet中，所以提供了`green.green`这个装饰器来包装我们需要处理异步IO的函数。

不同于其它支持Tornado的异步mysql驱动，greenlet完全没有改变pymysql的使用方式，也就是不需要再写yield，这样就为在Tornado中异步使用各种ORM提供了可能。

> <https://github.com/zhu327/greentor/tree/master/demo>

在demo中提供了一个在Tornado中使用Django ORM的实例，同时还支持Django admin对数据库进行管理。

### 总结

> <https://github.com/alex8224/gTornado>

greentor的大部分代码来自gTornado，感谢作者为我答疑解惑。下一步我会继续针对IOStream进行优化，以提升IOStream性能。
