---
date: 2015-09-29T18:36:08+08:00
title: Gunicorn运行Django时静态文件处理
tags: ["django"]
---

### 问题

在用Gunicorn跑Django的时候,比较郁闷的是静态文件的处理,即使在settings设置DEBUG=True,静态文件也不会正常显示.生产环境下一般不会裸跑Gunicorn,一般都会在前面放一个Nginx反代到Gunicorn,而静态文件直接交给Nginx处理.  

但是如heroku,coding.net的演示平台这种PaaS就不能自己配置反向代理,怎么样设置wsgi才能正常处理静态文件呢.这里总结下处理这个问题的经验.


### 方法1

强制使用Django的静态文件处理器,通过`python manage.py runserver`的时候,如果DEBUG=True,Django会自动加载自带的静态文件处理器,但是在Gunicorn下,这个设置会失效,我们可以强制使用Django自带的静态文件处理器.
<!--more-->

```python
from django.conf import settings
from django.conf.urls.static import static

urlpatterns = [
    # ... the rest of your URLconf goes here ...
] + static(settings.STATIC_URL, document_root=settings.STATIC_ROOT) # 附加static
```

### 方法2

在玩coding.net的演示平台的时候,发现了`Whitenoise`这个专门用来处理`wsgi app`静态文件的库,不用过多的设置,只要在app前加一个中间件就能正常处理静态文件了,并且还支持压缩缓存等强大的功能.  

参考[Deploying static files](http://python.usyiyi.cn/django/howto/static-files/deployment.html)先把Django下所有的静态文件归集到static目录下面.

在settings中做如下配置:

```python
STATIC_ROOT = os.path.join(os.path.dirname(__file__), '../forum/static')

STATIC_URL = '/static/'

STATICFILES_STORAGE = 'whitenoise.django.GzipManifestStaticFilesStorage' # 设置Gzip压缩
```

在wsgi.py下加上Whitenoise的中间件:

```python
import os

os.environ.setdefault("DJANGO_SETTINGS_MODULE", "xp.settings")

from django.core.wsgi import get_wsgi_application
application = get_wsgi_application()

from whitenoise.django import DjangoWhiteNoise # 这里import一定要放在os.environ后面,否则会报错
application = DjangoWhiteNoise(application)
```

然后愉快的运行Gunicorn吧,静态文件的问题就解决了.


### 总结

方法1虽然能用但是只是权益之际,毕竟Django自己的静态文件处理能力还是很弱的,所以推荐方法2,用Whitenoise搭配Gunicorn,完全能在没有Nginx的情况下在生产环境中处理静态文件.

### 参考

> <http://docs.coding.io/languages/python/>
> <http://whitenoise.readthedocs.org/en/latest/>
> <https://docs.djangoproject.com/en/1.5/howto/static-files/>
