---
date: 2014-09-24T22:13:57+08:00
title: Memcached 优化 MySQL 查询
tags: ["blog", "python", "memcached", "mysql"]
---

### 简介

[跬步](http://bozpy.sinaapp.com)上线后，SAE后台显示云豆的消耗http与MySQL查询各占一半，正好SAE还支持Memcached，本着不用白不用的原则，为[db.py](https://github.com/zhu327/blog/blob/memcached/www/transwarp/db.py)加上了Memcached的缓存支持。
 
网站在并发不大的情况下，MySQL查询时间还感受不出来，但是如果并发大的化，MySQL会显著的降低响应速度，所以Memcached应运而生。

Memcached是一个分布式的内存key-value存储系统，由于数据保存在内存中所以查询速度非常快，一般作为其它数据的中间缓存层来使用。

对于SQL查询先以SQL语句为key查询Memcached，如果查到直接返回，如果没有再去查询MySQL，并把结果缓存到Memcached，下次再次查询时速度显著提升，因为Memcached的数据是保存在内存中的。

<!--more-->
### 实现

```python
class Memcached(threading.local):
    def __init__(self):
        self._mc = lambda :pylibmc.Client() # 创建链接引擎，SAE的不需要输入IP:port，一般的形式为Client(['IP1:port', 'IP2:port'])，输入为一个Memcached的连接池
        self.con = None

    def _connect(self):
        if not self.con:
            self.con = self._mc()

    def clean(self):
        if self.con:
            self.con.disconnect_all()
            self.con = None

    def get(self, sql, args):
        key = self._key(sql, args)
        logging.info('MC get %s' % key)
        self._connect()
        return self.con.get(key)

    def set(self, sql, args, data):
        key = self._key(sql, args)
        logging.info('MC set %s' % key)
        self._connect()
        self.con.set(key, data) # 默认有效时间是30天，time=UTC失效时间，格式为unix标准时间

    def flush(self):
        self._connect()
        self.con.flush_all()

    def _key(self, sql, args): # 生成key:SQL语句的MD5值
        return hashlib.md5(sql % args).hexdigest()

mc = Memcached()
```
以上为pylibmc的简单封装，threading.local类的实例在不同的线程中是不同的，所以不同的线程会建立不同连接，互不干扰。

```python
def select(sql, *args):
    global _db_cxt
    sql = sql.replace('?', '%s')
    logging.info('select sql: %s, args: %s' % (sql, args))
    cursor = None
    try:
        r = mc.get(sql, args) # 先查询Memcached，查到直接返回
        if r:
            logging.info('MC get data')
            return r
        cursor = _db_cxt.cursor()
        cursor.execute(sql, args)
        if cursor.description:
            names = [x[0] for x in cursor.description]
        r = [dict(zip(names, x)) for x in cursor.fetchall()]
        mc.set(sql, args, r) # 缓存MySQL查询的结果，下次同样的查询就不必在查询MySQL
        return r
    finally:
        mc.clean()
        if cursor:
            cursor.close()
```
这里只针对SELECT语句，对于INSERT DELETE UPDATE语句，则执行清空所有缓存，以便生成新的缓存。

### 效果

使用缓存后Boz的响应时间因为没有并发没有明显提速的感觉，但是云豆的消耗比以前少了，MySQL的消耗少了很多。

### 参考

* [pylibmc docment](http://sendapatch.se/projects/pylibmc/)
* [Using MySQL and memcached with Python](http://dev.mysql.com/doc/mysql-ha-scalability/en/ha-memcached-interfaces-python.html)
* [python连接mysql，并在中间用memcached保存sql结果](http://www.cnblogs.com/valleylord/p/3505917.html)
