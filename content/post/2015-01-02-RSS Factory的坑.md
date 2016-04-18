---
date: 2015-01-02T19:41:03+08:00
title: RSS Factory的坑
tags: ["web", "tornado", "jinja2", "lxml"]
---

花了一天时间把[RSS Factory](https://github.com/zhu327/rss)更新了下，把Tornado默认的模版引擎换为jinja2，解析html第三方库换为lxml。遇到了几个坑，纪录下。

**自动转义**

Tornado默认的模版引擎默认自动转义，Flask配置的jinja2也自动转义了，[在Tornado中使用jinja2模版引擎的简单方法](http://bozpy.sinaapp.com/blog/15)，按照这个方法模版渲染的时候却没有自动转义，所以还需要进一步配置。

```python
application = tornado.web.Application(
template_loader=JinjaLoader(os.path.join(os.path.dirname(__file__), 'templates/'),
    autoescape=True, extensions=['jinja2.ext.autoescape']),
)
```
<!--more-->

`autoescape=True`开启jinja2的自动转义功能，`extensions=['jinja2.ext.autoescape'])`为jinja2增加扩展，在模版中可以控制某些不需要转义的段落关闭转义。

```python
{% autoescape false %}
    ...
{% endautoescape %}
```

jinja2的扩展功能非常强大，按照官方文档说明，除了自动转义还有很多可自定义的地方，暂时还没有需要。

**lxml中的字符串编码**

lxml fromstring tostring方法输出输入的字符串编码需要注意，如果字符串编码错误会导致jinja2渲染出问题。

```python
import lxml.etree, lxml.html # 直接import lxml 使用lxml.etree lxml.html不能用

lxml.etree.fromstring('''
<root></root>
''') # 这个函数只能接受string参数不能用unicode，所以只能把unicode先encode后才能正常使用

lxml.html.fromstring(unicode|string) # html可以接受unicode

lxml.html.tostring(xmlElement) # 输出的字符串为string，不是原来的unicode
lxml.html.tostring(xmlElement, encoding='unicode') # 输出unicdoe，encoding可以可以接受utf-8等编码格式参数
```

**xml解析**

微信公众号的xml始终不能用lxml.etree解析成功，应该是xml输出的不标准，郁闷的不行，又不想用BeautifulSoup，最后手写正则表达式搞定了，有时候还是需要一些笨方法。
