---
date: 2016-03-11T11:46:53+08:00
title: Django Model Trace 模型变更跟踪
tags: ["django"]
---

在公司业务中有很多模块需要记录模型的变更历史，刚开始的时候会针对需要变更记录的表增加一个变更记录表，然后做针对性的开发。后来类似的需求多了，而且不需要很详细的记录，所以单独提取了一个比较简单的模型跟踪模块出来。

### 原理

使用Django自己的信号实现，创建了一个公用的model跟踪表，所有配置跟踪的model变更记录都会保存到这张表中，通过一个公用的`trace_model`函数来保存变更的新老数据，而且只保存修改的字段。

```python
# coding=utf-8

import inspect, json
from django.conf import settings
from django.db.models import ManyToManyField, ManyToOneRel
from django.db.models.fields.related import ForeignKey
from raven.contrib.django.models import client
from models import ModelRecord


def trace_model(sender, instance, raw, using, update_fields, **kwargs):
    try:
        if instance._state.adding is True:
            return
        obj = sender.objects.get(pk=instance.pk)
        change_fields = {}
        for name in sender._meta.get_all_field_names():
            if name in ('add_by', 'add_dt', 'update_by', 'update_dt', 'update_by_id'): # 更新人，添加人，时间不记录
                continue
            field = sender._meta.get_field(name)
            if field.auto_created or field.hidden or field.__class__ in (ManyToManyField, ManyToOneRel): # 特殊的字段不记录
                continue
            old = getattr(obj, name, None)
            new = getattr(instance, name, None)
            if str(old) == str(new): # 不同类型的字段转为字符串，相等的不记录
                continue
            if isinstance(field, ForeignKey):# 如果是外键
                if name != field.name and not field.name in change_fields:
                    if old:
                        old = field.related_model.objects.filter(pk=old).first()
                    if new:
                        new = field.related_model.objects.filter(pk=new).first()
                elif name == field.name and not name+'_id' in change_fields:
                    pass # 判断避免同时对外键赋值的清空
                else:
                    continue
            if field.choices:
                if old:
                    old = getattr(obj, 'get_'+field.name+'_display', old)
                if new:
                    new = getattr(instance, 'get_'+field.name+'_display', new)
                if callable(old):
                    old = old()
                if callable(new):
                    new = new()
            old = str(old) if old is not None else '空'
            new = str(new) if new is not None else '空'
            verbose_name = getattr(field, 'verbose_name', name)
            change_fields[name] = {
                'name': verbose_name,
                'new': new,
                'old': old
            }
        dbname = getattr(sender, '_database', settings.DATABASES['default']['NAME'])
        for frame_record in inspect.stack():
            if frame_record[3] == 'get_response':
                request = frame_record[0].f_locals['request']
                user = request.user
                break
        # 数据入库
        if change_fields and 'user' in locals():
            ModelRecord.objects.create(dbname=dbname,
                                       table_name=sender._meta.db_table,
                                       table_pk = instance.pk,
                                       content=json.dumps(change_fields),
                                       add_by=user)
    except Exception:
        client.captureException()
```
<!--more-->

核心代码就是遍历Model的fields，比较新老数据，如果是外键，还需要记录Model的`__unicode__`方法返回的字符串，最后JSON序列话保存。

在获取`request`对象的时候，使用了`inspect.stack()`回溯了调用栈，直到找到操作人。

### 注册信号

在`settings.py`中设置`TRACE_CHANGE_MODEL`配置，比如像这样:

```python
TRACE_CHANGE_MODEL = [
    'apps.employee.models.EmployeeInfo',
    'apps.basic.insurance.weal_package.models.InsuranceWealPackage',
    'apps.basic.housing_fund.weal_package.models.HousingFundWealPackage',
    'apps.employee.models.DynamicFee',
]
```

在Django的APP环境加载完成后再注册需要跟踪的Model，避免出现重复相互import冲突

```python
# coding: utf-8

from django.apps import AppConfig
from django.utils.module_loading import import_string
from django.dispatch import receiver
from django.db.models.signals import pre_save
from django.conf import settings
from signals import trace_model


# 在Django加载完后注册信号处理
def ready(self):
    if hasattr(settings, 'TRACE_CHANGE_MODEL'):
        for name in settings.TRACE_CHANGE_MODEL:
            model = import_string(name)
            receiver(pre_save, sender=model, dispatch_uid='TRACE_CHANGE_MODEL')(trace_model)


AppConfig.ready = ready
```

### 源码

以上代码均已提交至github，仅供参考。

> <https://github.com/zhu327/model_trace>
