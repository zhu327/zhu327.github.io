---
date: 2015-03-11T19:36:41+08:00
title: SAE上用kvdb做Django缓存
tags: ["django"]
---

[Django forum](http://forum.sinaapp.com)在SAE上开启了5M的memcached缓存空间，memcached是用空间计费的，所以在没什么流量的情况下每天也要用1云豆，以blog上的经验来看不用缓存的情况下mysql的消耗又比较大，所以权衡之后选择SAE自己实现的kvdb缓存后端来做缓存，这样缓存消耗的云豆可以忽略不计，又可以达到缓存加速的效果。

参考Django自带的memcache缓存后端来写kvdb的缓存后端。在插入数据的时候添加一个超时时间戳，获取数据的时候判断数据是否超时，如超时则删除数据。另外在实现清空所有数据的时候kvdb没有实现相应的方法，我自己阅读kvdb文档后通过迭代key值来实现清空，有一个可能的问题是这个迭代删除最多只能一次删除100条数据。

### 实现

基于`django.core.cache.backends.base.BaseCache`，对kvdb已有的接口进行重写适配，kvdb未实现的接口则沿用。

> <https://github.com/zhu327/forum/blob/sae/xp/cache.py>

<!--more-->
```python
# coding: utf-8

"SAE kvdb cache backend"

import time
from threading import local

from django.core.cache.backends.base import BaseCache
from django.utils.encoding import force_str
from django.conf import settings

import sae.kvdb


class SaekvdbCache(BaseCache):
    "An implementation of a cache binding using sae kvdb"
    def __init__(self, server, params):
        super(SaekvdbCache, self).__init__(params)
        self._lib = sae.kvdb
        self._local = local()

    @property
    def _cache(self):
        # SAE kvdb应该也是C写的所以这里仿照PylibMC用线程局部名称空间
        client = getattr(self._local, 'client', None)
        if client:
            return client

        client = self._lib.KVClient(debug=settings.DEBUG)
        self._local.client = client

        return client

    def _get_timeout(self, timeout):
        """
        过期时间
        """
        timeout = timeout or self.default_timeout
        timeout += time.time()
        return timeout

    def make_key(self, key, version=None):
        # Python 2 memcache requires the key to be a byte string.
        return force_str(super(SaekvdbCache, self).make_key(key, version))

    def add(self, key, value, timeout=0, version=None):
        '''
        SAE kvdb不能自动过期，所以这里存储的时候多加一个时间戳
        '''
        key = self.make_key(key, version=version)
        obj = {
            'v': value,
            't': self._get_timeout(timeout)
        }
        return self._cache.add(key, obj)

    def get(self, key, default=None, version=None):
        key = self.make_key(key, version=version)
        val = self._cache.get(key)
        now = time.time()
        if val is None:
            return default
        elif val.get('t') < now: # 判断数据是否过期
            self._cache.delete(key)
            return default
        return val.get('v')

    def set(self, key, value, timeout=0, version=None):
        key = self.make_key(key, version=version)
        obj = {
            'v': value,
            't': self._get_timeout(timeout)
        }
        self._cache.set(key, obj)

    def delete(self, key, version=None):
        key = self.make_key(key, version=version)
        self._cache.delete(key)

    def get_many(self, keys, version=None):
        new_keys = [self.make_key(x, version=version) for x in keys]
        ret = self._cache.get_multi(new_keys)
        if ret:
            _ = {}
            m = dict(zip(new_keys, keys))
            now = time.time()
            for k, v in ret.items():
                if v.get('t') < now:
                    self._cache.delete(k)
                    continue
                _[m[k]] = v.get('v')
            ret = _
        return ret

    def close(self, **kwargs):
        self._cache.disconnect_all()

    def incr(self, key, delta=1, version=None):
        key = self.make_key(key, version=version)
        val = self._cache.get(key)
        now = time.time()
        if val is None:
            raise ValueError("Key '%s' not found" % key)
        elif val.get('t') < now:
            self._cache.delete(key)
            raise ValueError("Key '%s' not found" % key)
        new_value = val.get('v') + delta
        obj = {
            't': val.get('t'),
            'v': new_value
        }
        self._cache.set(key, obj)
        return new_value

    def clear(self):
        for k in self._cache.getkeys_by_prefix(''):
            self._cache.delete(k)
```

以上文件放在`Django project`目录下命名为`cache.py`，在`settings.py`中设置后即可。

```python
CACHES = { # 缓存设置
    'default': {
        'BACKEND': 'xp.cache.SaekvdbCache', # 可选用SAE kvdb做缓存，消耗云豆更少
        'LOCATION': '127.0.0.1:11211', # 这里的参数不起作用，但是为了方便切换缓存方式，这里保留这个设置
    }
}
```
