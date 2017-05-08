---
date: 2017-05-08T15:19:58+08:00
draft: true
title: zerorpc api设计指南
tags: ["zerorpc"]
---

> [Google API 设计指南](http://tailnode.tk/2017/03/google-api-design-guide/contents/)  
> [在Django中使用zerorpc](https://zhu327.github.io/2017/03/31/%E5%9C%A8django%E4%B8%AD%E4%BD%BF%E7%94%A8zerorpc/)

之前实现了在Django环境下使用zerorpc的封装,api的组织单位是function.受到Google API 设计指南的启发,重新设计了一种基于Resources的api组织风格,并且约定Resource的方法名与Django Rest Framework的ViewSets中实现的action名称一致.

zerorpc默认只能在同一个Server上面注册一个对象,而我们需要注册多个类(一个类表示一Resources),在阅读zerorpc Server的代码后,想到一个简单的方式来注册Resources.

<!--more-->

```python
import zerorpc

_methods = {}


def register(cls):
    obj = cls()
    prefix = cls.__name__
    _methods.update({'{}:{}'.format(prefix, k): getattr(obj, k)
                     for k in dir(obj)
                     if not k.startswith('_') and callable(getattr(obj, k))})
    return cls

if __name__ == '__main__':
    @register
    class Test(object):
        def add(self, x, y):
            return x + y

    server = zerorpc.Server(_methods, heartbeat=30)
    server.bind("tcp://{0}:{1}".format('127.0.0.1', 4242))
    server.run()
```
通过在方法名前增加类名称前缀在Server的`_methods`字典里面增加了`Class:method`格式的调用方法.这样就实现了注册多个类到zerorpc Server.

#### 约定

- 每个类代表一个Resources
- 方法名称**create,list,retrieve,update,destroy**分别对应于创建,列表,详情,更新,删除操作

约定尽量切合DRF已有的风格,不需要暴露的方法以`_`开头.

#### 调用

还需要改造下Client端,因为默认的Client不能直接使用`class:method`的方法名来调用rpc api

```python
import zerorpc

class Client(zerorpc.Client):
    def __init__(self, *args, **kwargs):
        self._prefix = kwargs.pop('prefix', '')
        super(Client, self).__init__(*args, **kwargs)

    def set_prefix(self, prefix):
        self._prefix = prefix

    def __getattr__(self, method):
        method = ':'.join([self._prefix, method]) if self._prefix else method
        return lambda *args, **kargs: self(method, *args, **kargs)
```

重写下Client,实例化时添加可选的prefix参数来指定调用的Resources,通过`set_prefix`方法来修改调用的Resources.
