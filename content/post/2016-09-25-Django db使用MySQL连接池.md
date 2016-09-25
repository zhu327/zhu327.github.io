---
date: 2016-09-25T16:41:33+08:00
title: Django db使用MySQL连接池
tags: ["django", "greentor"]
---

Django db模块本身不支持MySQL连接池，只有一个配置`CONN_MAX_AGE`连接最大存活时间，如果WSGI服务器使用了线程池技术，会达到连接复用的效果。但是如果WSGI服务如果是每个请求都创建新的线程，那么这个配置没有任何效果，因为连接保存在`Thread.local()`名称空间中，在不同的线程中不能复用。

在上一篇[greentor MySQL连接池实现](https://zhu327.github.io/2016/09/25/greentor-mysql连接池实现/)中已经实现了MySQL连接池，只需要重写Django MySQL backend以支持连接池，就能达到连接复用的目的，减少socket 3次握手的开销，提高性能。

<https://github.com/zhu327/greentor/blob/master/demo/core/base.py>

<!--more-->
```python
from django.db.backends.mysql.base import (SafeText, SafeBytes, six,
    DatabaseWrapper as BaseDatabaseWrapper)
from greentor.mysql import ConnectionPool

class DatabaseWrapper(BaseDatabaseWrapper):
    u"""
    支持greentor mysql connection pool 的backends
    """
    pools = {} # 类变量用于保存所有不同数据库的连接

    def get_new_connection(self, conn_params):
        # conn = Database.connect(**conn_params)
        if not self.alias in self.pools: # 如果需要的数据库还没有连接池则新建连接池
            self.pools[self.alias] = ConnectionPool(mysql_params=conn_params)
        conn = self.pools[self.alias].get_conn() # 获取新的连接时从连接池中获取
        conn.encoders[SafeText] = conn.encoders[six.text_type]
        conn.encoders[SafeBytes] = conn.encoders[bytes]
        return conn

    def _close(self):
        if self.connection is not None: # 不再直接关闭连接，而是释放连接到连接池中
            self.pools[self.alias].release(self.connection)
```

修改数据库配置引擎

```python
DATABASES = {
    'default': {
        'NAME': 'test',
        'HOST': '127.0.0.1',
        # 'ENGINE': 'django.db.backends.mysql',
        'ENGINE': 'core',
        'USER': 'root',
        'PASSWORD': '',
    }
}
```

连接池backend的代码虽然很少，但是在尝试过程中，基本把Django db模块的代码都过了一遍，感觉自己又牛B了一点点，哈哈。
