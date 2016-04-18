---
date: 2015-08-14T16:28:42+08:00
title: flask jinja2 i18n国际化简单实现
tags: ["jinja2"]
---

前两个星期,公司有需求要实现APP内嵌网页的英文版,因为要急着上线,所有就没有折腾i18n国际化,直接出了英文的模板,在view函数里面判断语言选择模板来实现.这周有空下来重新整了一下项目的国际化,使用通用的`gettext`来实现.

flask使用的是jinja2模板,jinja2自带了i18n扩展支持,但是需要Flask-Babel扩展的支持,有学习成本,所以没有使用,直接用了python内置的gettext来实现,而且jinja2引擎也提供了gettext的钩子来处理不同的语言.


## gettext

`gettext`是*nix下的标准i18n实现,Python的标准库内置`gettext`,简单的理解一下,就是通过已有的字典映射文件,然后获取环境的语言信息来判断当前使用哪一个语言映射,最终渲染出不同的语言文本,这里就不详细研究`gettext`了,提供我的参考文档:

> <http://underthehood.blog.51cto.com/2531780/1663604>
<!--more-->

## flask中使用jinja2

```python
app.jinja_env.add_extension('jinja2.ext.i18n') # 设置jinja2 env,增加'jinja2.ext.i18n'扩展
```

判断请求语言环境,设置语言的上下文:

```python
import os, gettext
from flask import current_app

i18n_dir = os.path.join(os.path.abspath(os.path.dirname(__file__)), 'frontend/i18n') # gettext语言映射文件的目录

def set_lang(lang):
    #'lang'表示语言文件名为lang.mo，'i18n'表示语言文件名放在‘i18n'目录下，比如：
    #中文翻译目录和文件：i18n\zh-cn\LC_MESSAGES\lang.mo
    gettext.install('lang', i18n_dir, unicode=True)
    tr = gettext.translation('lang', i18n_dir, languages=[lang])
    tr.install(True) # 设置语言
    current_app.jinja_env.install_gettext_translations(tr) # 通过jinja2的钩子设置模板中使用的语言环境
```

这里的`set_lang`只是用来辅助设置语言信息的,可用在请求前设置:

```python
@app.before_request
def before_request():
    lang = request.args.get('lang', None) # 我这里用的QueryString来放语言参数,实际应用中最好从request头的HTTP_ACCEPT_LANGUAGE取
    set_lang(lang)
```

当然也可用通过中间件的形式来实现,我们怎么简单怎么来.这样在模板中就可用通过下面这种形式来实现多语言支持了.

```python
{{_("English")}}
{%trans%}English{%endtrans%}
```

以上flask国际化方案都是源自网络,主要参考了一下内容:

> <http://underthehood.blog.51cto.com/2531780/1663604>
> <http://bbs.chinaunix.net/thread-4094001-1-1.html>
