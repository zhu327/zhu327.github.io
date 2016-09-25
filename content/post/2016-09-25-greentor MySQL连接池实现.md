---
date: 2016-09-25T15:56:21+08:00
title: greentor MySQL连接池实现
tags: ["tornado", "django", "greentor"]
---

> <https://en.wikipedia.org/wiki/Connection_pool>

通过[greentor](https://github.com/zhu327/greentor)实现了pymysql在Tornado上异步调用的过程后发现，每次建立数据库连接都会经过socket 3次握手，而每一次socket读写都会伴随着greenlet的切换，以及ioloop的callback过程，虽然是异步了，但是IO性能并没有提升，所以在研究了[TorMySQL](https://github.com/snower/TorMySQL)连接池的实现后，实现了greentor自己的连接池。

<https://github.com/zhu327/greentor/blob/master/greentor/green.py>

<!--more-->
```python
class Pool(object):
    def __init__(self, max_size=32, wait_timeout=8, params={}):
        self._maxsize = max_size # 连接池最大连接数
        self._conn_params = params # 连接参数
        self._pool = deque(maxlen=self._maxsize) # 存储连接的双端队列
        self._wait = deque() # 等待获取连接的callback
        self._wait_timeout = wait_timeout # 等待超时时间
        self._count = 0 # 已创建的连接数
        self._started = False # 连接池是否可用
        self._ioloop = IOLoop.current()
        self._event = Event() # 连接池关闭时间，set该时间后，连接池所有的连接关闭
        self._ioloop.add_future(spawn(self.start), lambda future: future) # 在greenlet中启动连接池

    def create_raw_conn(self):
        pass # 通过self._conn_params参数创建新连接，用于重写

    def init_pool(self): # 创建新的连接，并加入到连接池中
        self._count += 1
        conn = self.create_raw_conn()
        self._pool.append(conn)

    @property
    def size(self): # 可用的连接数
        return len(self._pool)

    def get_conn(self):
        while 1:
            if self._pool: # 如果有可用连接，直接返回
                return self._pool.popleft()
            elif self._count < self._maxsize: # 如果没有可用连接，且创建的连接还没有达到最大连接数，则新建连接
                self.init_pool()
            else:
                self.wait_conn() # 如果没有可用连接，且以创建了最大连接数，则等待连接释放

    def wait_conn(self):
        timer = None
        child_gr = greenlet.getcurrent()
        main = child_gr.parent
        try:
            if self._wait_timeout: # 创建计时器，如果等待了超时则抛出异常
                timer = Timeout(self._wait_timeout)
                timer.start()
            self._wait.append(child_gr.switch)
            main.switch() # 切换到父greenlet上，直到child_gr.switch被调用
        except TimeoutException, e:
            raise Exception("timeout wait connections, connections size %s", self.size)
        finally:
            if timer:
                timer.cancel()

    def release(self, conn):
        self._pool.append(conn) # 释放连接，重新加入连接池中
        if self._wait: # 如果有等待的greenlet
            callback = self._wait.popleft()
            self._ioloop.add_callback(callback) # 在下次ioloop过程中切换到等待获取连接的greenlet

    def quit(self): # 关闭连接池
        self._started = False
        self._event.set()

    def _close_all(self):
        for conn in tuple(self._pool):
            conn.close()
        self._pool = None

    def start(self):
        # self.init_pool()
        self._started = True
        self._event.wait()
        self._close_all()
```

这是一个通用连接池，通过继承`Pool`类，并重写`create_raw_conn`方法就可用实现一个简单的连接池，比如mysql，memcache等

<https://github.com/zhu327/greentor/blob/master/greentor/mysql.py>

```python
from pymysql.connections import DEBUG, Connection

class ConnectionPool(Pool):
    def __init__(self, max_size=32, keep_alive=7200, mysql_params={}):
        super(ConnectionPool, self).__init__(max_size=max_size, params=mysql_params)
        self._keep_alive = keep_alive # 为避免连接自动断开，配置连接ping周期

    def create_raw_conn(self):
        conn = Connection(**self._conn_params)
        if self._keep_alive:
            self._ioloop.add_timeout(time.time()+self._keep_alive, self._ping, conn)
        return conn

    @green
    def _ping(self, conn):
        if conn in self._pool:
            self._pool.remove(conn)
            conn.ping()
            self.release(conn)
        self._ioloop.add_timeout(time.time()+self._keep_alive, self._ping, conn)
```

这里实现了一个MySQL的连接池，MySQL默认会有一个8小时空闲，连接自动断开的机制，为了避免连接池中出现无效连接，在设置了`keep_alive`时间后会周期性的调用连接的`ping`方法来保证连接的活跃，同时如果连接不在连接池中，说明连接正在被使用，就不需要再`ping`了。

事隔三个月，在一位新认识的朋友提醒下，又捡起了greentor的开发，把之前准备实现的连接池写了出来。
