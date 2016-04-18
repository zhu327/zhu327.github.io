---
date: 2014-09-15T19:36:00+08:00
title: Python 使用 MySQL
tags: ["python", "MySQL"]
---

### MySQLdb基本使用方式

```python
import MySQLdb

# 建立MySQL连接
connect = MySQL.connect(user=user, passwd=passwd, db=db, host=host, port=port)
cursor = connect.cursor() # 生成游标对象

try:
    cursor.execute('SQL %s', args) # 游标对象执行SQL语句，语句中有%s，则用args替换

    # 执行SELECT语句通过以下方法可获取到查询结果
    result = cursor.fetchall() # 结果为列表，列表每一项代表查询的一row，每行都是一个列表，列表顺序按照SELECT的字段顺序排序
    result = cursor.fetchone() # 获取查询到结果的第一行数据，返回列表，再次运行获取下一条

    cursor.description # 查询到字段定义信息

    [x[0] for x in cursor.description] # 序列的第一个值为字段名称

    # 执行INSERT DELETE操作获取受影响的行数
    r = cursor.rowcount

    # 获取INSERT行的主键值，一般是id
    id = cursor.lastrowid

    connect.commit() # 游标对象执行语句是以事务方式执行，所以需要提交
except: # 如果提交失败
    connect.rollback() # 回滚事务
finnally:
    # 最后关闭游标，连接
    cursor.close()
    connect.close()
```
<!--more-->

### 封装思路

从用户角度来设计使用方式，然后再设计实现方式，用户可以这样来用db.py

```python
from transwarp import db

# 创建数据库链接引擎，然后就可以直接调用数据操作
db.create_engine(user='root', password='password', host='host', port=3306, database='test')

# 直接操作数据查询，返回列表，列表元为字典为查询的每一行数据
user = db.select('select * from user')
# user =>
# [
#     {'id': 1, 'name': 'Timmy Chu'},
#     ......
# ]

# 执行INSERT DELETE UPDATE返回受影响的行数
n = db.update('insert int user(id, name) value(?, ?)', 4, 'Jack')
# n =>
# 1
```
具体实现代码在这里[db.py](https://github.com/zhu327/boz/blob/master/www/transwarp/db.py)

以下为重点:

1. create_engine函数会生成一个全局变量engine，engine为持有数据库建立*函数*的对象(_Engine类的实例);
2. _db_cxt全局变量持有该数据库引擎，并且建立游标，以及关闭链接的函数
3. _ConnectionCxt上下文类，`__enter__()`方法判断是否已经建立链接，没有则执行初始化数据库链接，并把是否需要清理置为`True`，`__exit__()`判断是否需要执行清理，需要则关闭连接
4. 对于事件在_db_cxt全局变量中保存一个transactions变量记录执行语句的条数，只有条数为0才提交

根据以上提示再阅读代码，就很容易理解了。  
另外可以参考廖雪峰的教程: [编写数据库模块](http://www.liaoxuefeng.com/wiki/001374738125095c955c1e6d8bb493182103fac9270762a000/0013976160374750f95bd09087744569be5aae6160c8351000)
