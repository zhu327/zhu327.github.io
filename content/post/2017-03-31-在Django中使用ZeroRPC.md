---
date: 2017-03-31T15:59:13+08:00
title: 在Django中使用zerorpc
tags: ["django", "zerorpc"]

---

### 前言

随着系统架构从集中式单点服务器到分布式微服务方向的迁移,RPC是一个不可回避的话题.如何在系统中引入对开发者友好,性能可靠的RPC服务是一个值得深思的问题.

在调研了Thrift,gRPC,zerorpc等方案后,基于以下2点最后选择了zerorpc:

- Thrift,gRPC学习成本高,开发者需要重新定义返回结构增加了工作量
- zerorpc完美契合Python,能快速开发,并且支持Node.js,适用于当前技术栈

### 问题

虽然zerorpc可以直接嵌入当前系统框架中,但是还是有一些问题需要去考虑解决

- rpc 接口如何定义

- rpc 服务如何启动

- 高并发情况下客户端的可靠性

<!--more-->

### 服务端

在当前的系统中大量使用Celery,djang-celery定义Task的方式是在每个install app中定义`tasks.py`文件,然后通过`@task`装饰器来生成Task.所以这里为了方便定义rpc interface设计一套类似于Celery的规范.需要输出rpc interface的app下面创建`rpcs.py`文件

```python
# rpcs.py
# coding: utf-8

from eebo.core.utils.zrpc import rpc
from .models import Ticket
from .serializers import TicketSerializer


@rpc.register()
def get_ticket():
    t = Ticket.objects.first()
    s = TicketSerializer(t)
    return s.data


@rpc.register(name='ticket_list', stream=True)
def get_tickets(n):
    qs = Ticket.objects.all()[:n]
    s  = TicketSerializer(qs, many=True)
    return iter(s.data)
```

`rpc.register`装饰器用来注册函数到rpc服务上,可选参数:

- name: 客户调用方法名称, 没有写的情况下就是func name如get_ticket
- stream: 默认False, 如果为True, 则使用zerorpc的流式响应传输, 数据量比较大的情况时使用, 返回可迭代对象

我们来看看`eebo.core.utils.zrpc`如何来实现这个注册过程:

```python
# coding: utf-8

import zerorpc


class RPC(object):
    @classmethod
    def register(cls, name=None, stream=False):
        def _wrapper(func):
            setattr(cls, name or func.__name__, zerorpc.stream(
                lambda self, *args, **kwargs: func(*args, **kwargs)) if stream
                    else staticmethod(func))
            return func

        return _wrapper


rpc = RPC()
```
通过一个类方法来往类上面绑定方法,需要注意的是`name`的定义必须是全局唯一的.

现在我们有了定义rpc interface的方法,下面来看看如何启动rpc server.

```python
# runrpc.py
# coding: utf-8

import re
import sys
import imp as _imp
import importlib
from django.conf import settings
from django.core.management.base import BaseCommand, CommandError

from eebo.core.utils.zrpc import rpc, ServerExecMiddleware

naiveip_re = re.compile(r"""^(?:
(?P<addr>
    (?P<ipv4>\d{1,3}(?:\.\d{1,3}){3}) |         # IPv4 address
    (?P<ipv6>\[[a-fA-F0-9:]+\]) |               # IPv6 address
    (?P<fqdn>[a-zA-Z0-9-]+(?:\.[a-zA-Z0-9-]+)*) # FQDN
):)?(?P<port>\d+)$""", re.X)


class Command(BaseCommand):
    help = "Starts a lightweight RPC server for development."

    default_addr = '127.0.0.1'
    default_port = '4242'

    def add_arguments(self, parser):
        parser.add_argument('addrport',
                            nargs='?',
                            help='Optional port number, or ipaddr:port')

    def handle(self, *args, **options):

        self.use_ipv6 = False
        if not options['addrport']:
            self.addr = ''
            self.port = self.default_port
        else:
            m = re.match(naiveip_re, options['addrport'])
            if m is None:
                raise CommandError('"%s" is not a valid port number '
                                   'or address:port pair.' %
                                   options['addrport'])
            self.addr, _ipv4, _ipv6, _fqdn, self.port = m.groups()
            if not self.port.isdigit():
                raise CommandError("%r is not a valid port number." %
                                   self.port)
            if self.addr:
                if _ipv6:
                    self.addr = self.addr[1:-1]
                    self.use_ipv6 = True
                    self._raw_ipv6 = True
                elif self.use_ipv6 and not _fqdn:
                    raise CommandError('"%s" is not a valid IPv6 address.' %
                                       self.addr)
        if not self.addr:
            self.addr = self.default_addr_ipv6 if self.use_ipv6 else self.default_addr
            self._raw_ipv6 = self.use_ipv6
        self.run(**options)

    def run(self, **options):
        """Run the server, using the autoreloader if needed."""
        self.autodiscover_rpc()

        server = self.get_server()

        try:
            server.run()
        except KeyboardInterrupt:
            server.close()
            sys.exit(0)

    def autodiscover_rpc(self, related_name='rpcs'):
        for pkg in settings.INSTALLED_APPS:
            try:
                pkg_path = importlib.import_module(pkg).__path__
            except AttributeError:
                continue

            try:
                _imp.find_module(related_name, pkg_path)
            except ImportError:
                continue

            try:
                importlib.import_module('{0}.{1}'.format(pkg, related_name))
            except ImportError:
                pass

    def get_server(self, *args, **options):
        """Return the default zerorpc server for the runner."""
        import zerorpc
        server = zerorpc.Server(rpc, heartbeat=30)
        server.bind("tcp://{0}:{1}".format(self.addr, self.port))
        # close django old connections
        zerorpc.Context.get_instance().register_middleware(ServerExecMiddleware())

        # for sentry
        try:
            from raven.contrib.zerorpc import SentryMiddleware
            if hasattr(settings, 'RAVEN_CONFIG'):
                sentry = SentryMiddleware(hide_zerorpc_frames=False,
                                          dsn=settings.RAVEN_CONFIG['dsn'])
                zerorpc.Context.get_instance().register_middleware(sentry)
        except ImportError:
            pass

        return server
```

`runrpc.py`是一个Django management commands 文件需要放到某个install app目录的`management/commands`下面,启动服务器:

```shell
python manage.py runrpc 0.0.0.0:4242
```
- `autodiscover_rpc` 自动发现rpc interface注册函数
- `get_server` 生成zerorpc server对象

在`get_server`中对zerorpc注册了2个中间件,`SentryMiddleware`用于捕获rpc interface抛出的异常发送到sentry,`ServerExecMiddleware`用于处理Django db connection,看看代码:

```python
# zrpc.py
# coding: utf-8

from django.db import close_old_connections

class ServerExecMiddleware(object):

    def server_before_exec(self, request_event):
        close_old_connections()

    def server_after_exec(self, request_event, reply_event):
        close_old_connections()
```

在每个rpc interface被调用前与调用后都调用`close_old_connections`关闭db connection,这里是为了实现`django.db`中对请求处理前与处理后注册信号:

`django.db.__init__.py`

```python
signals.request_started.connect(close_old_connections)
signals.request_finished.connect(close_old_connections)
```

目的是保证在rpc interface中使用ORM时,connection没有超时断开.

### 客户端

由于rpc的调用是阻塞的,不能全局只创建一个client.但是也不能每个请求都创建client,所以这里参考`redis-py`的client实现,定义一个支持连接池的zerorpc client.

```python
# zrpc.py
# coding: utf-8

import os
import zerorpc

from redis.connection import BlockingConnectionPool
from gevent.queue import LifoQueue

class Connection(object):
    def __init__(self, connect_to, heartbeat=30):
        self.client = zerorpc.Client(heartbeat=heartbeat)
        self.client.connect(connect_to)
        self.pid = os.getpid()

    def disconnect(self):
        self.client.close()


class RPCClient(object):
    def __init__(self, connect_to, heartbeat=30):
        self.connection_pool = BlockingConnectionPool(connection_class=Connection,
            queue_class=LifoQueue, timeout=heartbeat, connect_to=connect_to, heartbeat=heartbeat)

    def close(self):
        self.connection_pool.disconnect()

    def __getattr__(self, name):
        return lambda *args, **kwargs: self(name, *args, **kwargs)

    def __call__(self, name, *args, **kwargs):
        connection = self.connection_pool.get_connection('')
        try:
            return getattr(connection.client, name)(*args, **kwargs)
        finally:
            self.connection_pool.release(connection)
```
这里直接复用了`redis-py`定义的连接池,当前系统使用gunicorn + gevent的方式启动Django服务,所以`queue_class`使用了gevent的`LifoQueue`.

在使用过程中还发现了这个问题:

> https://github.com/0rpc/zerorpc-python/issues/123

需要打个补丁解决:

```python
import zmq.green as zmq

# patch zmq garbage-collection Thread to use green Context:
from zmq.utils.garbage import gc
gc.context = zmq.Context()
```

### 总结

技术的选型需要契合项目实际情况,不要盲目上新技术引入不必要的成本.为了推广方案,必须全局的考虑方案是否易使用,是否易部署.

完整代码:

> https://gist.github.com/zhu327/5b6c06eccc5758d4e642ee899a518687
