---
date: 2016-05-16T09:49:32+08:00
title: Django save方法的update_fields参数
tags: ["django"]
---

在Django中对Model对象的属性赋值，并使用save方法，save方法并不会检测Model对象变更了什么属性，而是会使用`update`语句set所有的field为当前的值，也就是说只修改了部分字段，也会update所有字段的值，这样会导致2个问题:

1. SQL语句过大，导致SQL执行慢(一般也不会有什么影响)
2. 如果同时操作同一个Model对象，更新不同的字段，后save的操作会用老的值覆盖掉早点操作的值

所以推荐在调用save方法的时候传`update_fields`参数，但是这样就要求我们必须对每次save的fields都在代码中记下来，不是很方便，所以开发一个通用的专门用来处理update_fields参数的Mixin类来解决这个问题。

<!--more-->

```python
class ModelUpdateFieldsMixin(object):
    '''
    Model对象在save的时候如果没有传update_fields参数，则自动跟踪对象字段的修改，并且在save的时候自动加上update_fields
    '''
    def __init__(self, *args, **kwargs):
        super(ModelUpdateFieldsMixin, self).__init__(*args, **kwargs)

        self._changed_fields = {} # 初始化字典，用户存储字段变更的信息

    def __setattr__(self, name, value):
    	'''
        重写__setattr__方法用于对属性赋值检测
        '''

        if hasattr(self, '_changed_fields'):
            try:
                field = self._meta.get_field(name) # 通过属性名称获取field对象，如果不是field属性则忽略
            except FieldDoesNotExist:
                field = None
            if field and not (field.auto_created or field.hidden) and field.__class__ not in (ManyToManyField, ManyToOneRel): # 如果不是自动创建，隐藏字段，多对多，多对一的关联对象
                try:
                    old = getattr(self, name, DoesNotExist) # 获取赋值前的属性值
                except field.rel.to.DoesNotExist:
                    old = DoesNotExist

                super(ModelUpdateFieldsMixin, self).__setattr__(name, value) # 赋值
                new = getattr(self, name, DoesNotExist) # 获取新的属性值
                try:
                    changed = (old != new) # 比较
                except:
                    changed = True
                if changed: # 如果发现值变更了
                    changed_fields = self._changed_fields
                    if name in changed_fields: # 如果已经记录过且值与新的值一样，则忽略
                        if changed_fields[name] == new:
                            changed_fields.pop(name)
                    else:
                        changed_fields[name] = copy(old) # 记录老的值
            else:
                super(ModelUpdateFieldsMixin, self).__setattr__(name, value)
        else:
            super(ModelUpdateFieldsMixin, self).__setattr__(name, value)

    def save(self, *args, **kwargs):
        '''
        重写save方法，传update_fields参数
        '''
        if not self._state.adding and hasattr(self, '_changed_fields') and 'update_fields' not in kwargs and not kwargs.get('force_insert', False):
            kwargs['update_fields'] = [key for key, value in self._changed_fields.iteritems() if hasattr(self, key)]
            self._changed_fields = {}
        return super(ModelUpdateFieldsMixin, self).save(*args, **kwargs)
```

这样在定义Model类的时候加上这个Mixin就不必在使用save的方法的时候传update_fields，会自动检测并生成update_fields。
