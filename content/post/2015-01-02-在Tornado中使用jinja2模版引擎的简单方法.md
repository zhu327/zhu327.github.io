---
date: 2015-01-02T10:34:28+08:00
title: 在Tornado中使用jinja2模版引擎的简单方法
tags: ["web", "jinja2", "tornado"]
---

### 参考

* [Tornado使用Jinja2模板引擎](http://veerkat.me/blog/posts/Tornado-shi-yong-Jinja2-mo-ban-yin-qing.html)

### 实现

为了让自己的开发工具都能得到统一，所以想着以后不管是用Tornado也好，Bottle也好，模版引擎都开始用jinja2，所以更新了[RSS Factory](https://github.com/zhu327/rss)使用的模版引擎。
 
Google了下Tornado使用jinja2的方法，发现大部分人的实现方法都是重写`ReaquestHandle`的`render`方法，这样的虽然比较简单但是不利于保持Tornado的完整性，所以我使用参考文章中的方法。

<!--more-->
通过查看代码发现，Tornado在渲染模版的时候会查找`settings["template_loader"]`这个实例，如果没有就用默认模版渲染，所以要做的就是在创建`tornado.web.Application`的时候定义`template_loader`。

```python
application = tornado.web.Application(
    template_loader=JinjaLoader(os.path.join(os.path.dirname(__file__), 'templates/')),
)
```

可以看到定义了`template_loader`并且以模版的目录为参数，接下来实现`JinjaLoader`。

```python
import threading
from tornado import template, web
import jinja2

class TTemplate(object):
    def __init__(self, template_instance):
        self.template_instance = template_instance

    def generate(self, **kwargs):
        return self.template_instance.render(**kwargs)

class JinjaLoader(template.BaseLoader):
    def __init__(self, root_directory, **kwargs):
        self.jinja_env = \
        jinja2.Environment(loader=jinja2.FileSystemLoader(root_directory), **kwargs)
        self.templates = {}
        self.lock = threading.RLock()

    def resolve_path(self, name, parent_path=None):
        return name

    def _create_template(self, name):
        template_instance = TTemplate(self.jinja_env.get_template(name))
        return template_instance
```

`JinjaLoader`继承自`template.BaseLoader`，通过`_create_template`方法最终把模版实例传给Tornado，但是Tornado中最后对模版使用字典渲染的时候使用的是`generate`方法，
而不是Jinja2默认的`render`方法，所以还需要`TTemplate`类来封装`render`这个过程。  
这样我们就使用了tornado默认的接口而没有修改Tornado的模版实现逻辑来在Tornado中使用jinja2模版。
