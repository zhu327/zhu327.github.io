---
date: 2015-01-10T19:28:37+08:00
title: Django实现JOIN查询
tags: ["django"]
---

在写Django Model的Manager面临的一大问题是怎么样才能实现Django Model的复杂查询，比如LEFT JOIN语句的使用，Google用了不少时间，基本上了解大概，总结下，大概有3种方式。

### 参考

> [闲话 Django Raw SQL](http://wing2south.com/post/django-raw-sql/)  
> [实例详解Django的 select_related 和 prefetch_related 函数对 QuerySet 查询的优化](http://blog.jobbole.com/74881/)

### 想法

写Model的时候参考<http://f2m.im>的表结构，发现它大量使用了JOIN查询，而且在多个表上都没有使用外键，我想不用外键也是为了方便修改，删除时产生不必要的约束。Django Model实现了ForeignKey，很方便的能查询到外键所在的实例，进一步能查询的更多，但是查询一次相当于执行一次SQL语句，这就造成了性能开销。所以F2E才会使用JOIN语句把计算的过程都交给MySQL。
<!--more-->

### 实现

1. 使用ForeignKey与select_related

    ```python
    Topic.objects.select_related('author').all() # 查询并且查询auhtor的信息，通过 LEFT.JOINT author 表实现
    ```

    这种方式依赖ForeignKey，所以需要删除数据的时候要小心，避免产生关联错误。

2. Manager.raw()

    ```python
    Topic.objects.raw('select * form forum_topic')
    ```

    直接执行SQL查询语句，这就比较方便了，也不需要外键来约束，但是直接写SQL好像又不能体现ORM的优越性。

3. DB-API

    ```python
    from django.db import connection

    cursor = connection.cursor()
    ```

这就完全脱离ORM的范围了，直接使用了MySQLbd的api，不推荐使用这种方式。

以上3种方式都不完美，需要取舍，能很好的定义外键的情况下使用第一种方式最好。
