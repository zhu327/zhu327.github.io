---
date: 2014-09-14T15:08:34+08:00
title: Python with语句
tags: ["python"]
---

### 参考

[浅谈 Python 的 with 语句](http://www.ibm.com/developerworks/cn/opensource/os-cn-pythonwith/)

### 概念

上下文管理器:  
实现了`__enter__()`与`__exit__()`方法的类实例，运行时，先运行`__enter__()`方法，再运行目标函数，无论是否抛出错误，都运行`__exit__()`方法。  
with语句可以方便的取代`try:/except:`语句达到在运行函数前先构建环境，退出函数清理环境的目的。  

<!--more-->
```python
with open('filename') as somefile:
    for line in somefile:
        pass
```

打开文件把文件对象赋值给somefile，操作后自动关闭文件，无论操作是否抛错。  
使用方法:

```python
with context_expression [as target(s)]:
    with body
```
`target(s)`是上下文对象`__enter__()`方法的返回值，多个返回值需要用tuple，返回值可在with body中操作。

### 自定义上下文管理器

```python
class Context(object):
    def __init__(self, ):
        pass
    def __enter__(self, ):
        pass
    def __exit__(self, ):
        pass
```
这里要注意的是`with Context()`使用的时候with后面是Context的实例。

### 使用装饰器操作上下文管理器

```python
import functools

def with_context(func):
    @functools.wraps(func)
    def _wrapper(*args, **kw):
        with Context():
            return func(*args, **kw)
        return _wrapper
```
### 补充: 关于异常捕获

```python
    # 抛出异常
    raise errorTyepe, value, traceback # 异常类型，异常值，异常信息
    # 捕获异常
    try:
        pass
    except Error, e:
        print e # 捕获到错误类型为Error的异常，e为错误的traceback

    # with语句中的异常
    __exit__(self, type, value, traceback):
    # 捕获到with body中产生的错误，在退出时根据错误做处理
```
