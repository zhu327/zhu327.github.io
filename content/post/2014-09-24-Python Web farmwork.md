---
date: 2014-09-24T20:52:31+08:00
title: Python Web farmwork
tags: ["python", "web"]
---

### 简介

* [编写Web框架](http://www.liaoxuefeng.com/wiki/001374738125095c955c1e6d8bb493182103fac9270762a000/0014023080708565bc89d6ab886481fb25a16cdc3b773f0000)
* [web.py](https://github.com/zhu327/blog/blob/master/www/transwarp/web.py)

[上一节](http://bozpy.sinaapp.com/blog/7)写了WSGI与Cookie相关的内容，为了方便的实现Web App，需要编写一个简单的Web框架，需要实现以下功能:

1. 处理request；
2. 生成response header；
3. 根据request URI找到对应处理的函数，即URL路由；
4. 定制模版引擎；
5. 拦截器，在处理函数产生响应前，做预处理，相当于WSGI中middlware的概念，比如处理Cookie等。

<!--more-->
### Request

```python
class Request(object):

    def __init__(self, environ): # 参数为WSGI中的environ
        self._environ = environ

    def _get_raw_input(self):
        '''
        用来生成Request的k-v字典
        wsgi.input 类文件对象，保存了post请求中的body
        cgi.FieldStorage() 生成字典，处理包括environ中的QueryString和wsgi.input中的参数，格式化
        '''
        if not hasattr(self, '_raw_input'):
            def _convet(item):
                if isinstance(item, list):
                    return [_to_unicode(i.value) for i in item]
                if item.filename:
                    return MultiFile(item)
                return _to_unicode(item.value)
            fs = cgi.FieldStorage(fp=self._environ['wsgi.input'], environ=self._environ, keep_blank_values=True)
            d = {}
            for k in fs.keys():
                d[k] = _convet(fs[k])
            self._raw_input = d
        return self._raw_input

    # 根据key返回value
    def get(self, key, default=None):
        return self._get_raw_input().get(key, default)

    # 返回key-value的dict
    def input(self, **kw):
        '''
        返回字典的拷贝
        '''
        copy = dict(**kw)
        for k,v in self._get_raw_input().iteritems():
            copy[k] = v[0] if isinstance(v, list) else v
        return copy

    # 返回URL的path
    @property
    def path_info(self):
        return urllib.unquote(self._environ.get('PATH_INFO', ''))

    # 返回request method
    @property
    def method(self):
        return self._environ['REQUEST_METHOD']

    def _get_headers(self):
        '''
        获取HTTP头，在Request中以HTTP_开头的都是header
        '''
        if not hasattr(self, '_headers'):
            d = {}
            for k,v in self._environ.iteritems():
                if k.startswith('HTTP_'):
                    d[k[5:].replace('_', '-')] = _to_unicode(v)
            self._headers = d
        return self._headers

    # 返回HTTP header
    @property
    def headers(self):
        return dict(**self._get_headers())

    def _get_cookie(self):
        '''
        获取Cookie，HTTP头中HTTP_COOKIE，格式化Cookie返回字典
        '''
        if not hasattr(self, '_cookie'):
            d = {}
            c_str = self._environ.get('HTTP_COOKIE')
            if c_str:
                for c in c_str.split(';'):
                    pos = c.find('=')
                    d[c[:pos].strip()] = urllib.unquote(_to_unicode(c[pos+1:]))
            self._cookie = d
        return self._cookie

    # 更加key返回Cookie value
    def cookie(self, name, default=None):
        return self._get_cookie().get(name, default)

    @property
    def cookies(self):
        return dict(**self._get_cookie())
```

### Response header

```python
class Response(object):
    def __init__(self):
        self._status = '200 OK'
        self._headers = {'CONTENT-TYPE': 'text/html; charset=utf-8'}

    @property
    def headers(self):
        '''
        生成header，设置Cookie，Set-Cookie
        _RESPONSE_HEADER_DICT是一个保存了常用header的字典
        '''
        L = [(_RESPONSE_HEADER_DICT.get(k, k), v) for k, v in self._headers.iteritems()]
        if hasattr(self, '_cookie'):
            for v in self._cookie.itervalues():
                L.append(('Set-Cookie', _to_str(v)))
        L.append(_HEADER_X_POWERED_BY)
        return L

    # 设置header
    def set_header(self, name, value):
        key = name.upper()
        if not key in _RESPONSE_HEADER_DICT:
            key = name
        self._headers[key] = _to_str(value)

    def unset_header(self, name):
        key = name.upper()
        if not key in _RESPONSE_HEADER_DICT:
            key = name
        if key in self._headers:
            del self._headers[key]

    @property
    def content_type(self):
        return self._headers.get('CONTENT-TYPE', None)

    @content_type.setter
    def content_type(self, value):
        if value:
            self.set_header('CONTENT-TYPE', value)
        else:
            self.unset_header('CONTENT-TYPE')

    # 设置Cookie
    def set_cookie(self, name, value, max_age=None, expires=None, path='/'):
        '''
        生成Cookie
        '''
        if not hasattr(self, '_cookie'):
            self._cookie = {}
        l = ['%s=%s' % (urllib.quote(name), urllib.quote(value))]
        if expires:
            if isinstance(expires, (float, int, long)):
                l.append('Expires=%s' % datetime.datetime.fromtimestamp(expires, UTC_0).strftime('%a, %d-%b-%Y %H:%M:%S GMT'))
            if isinstance(expires, (datetime.datetime, datetime.time)):
                l.append('Expires=%s' % expires.astimezone(UTC_0).strftime('%a, %d-%b-%Y %H:%M:%S GMT'))
        if isinstance(max_age, (int, long)):
            l.append('Max_Age=%s' % max_age)
        l.append('Path=%s' % path)
        self._cookie[name] = '; '.join(l)

    def delete_cookie(self, name):
        '''
        删除Cookie实际就是把过期时间设置到当前时间之前
        '''
        self.set_cookie(name, '__delete__', expires=0)

    # 设置status
    @property
    def status(self):
        return self._status

    @status.setter
    def status(self, value):
        '''
        设置status
        _RESPONSE_STATUSES status的字典
        '''
        if isinstance(value, (int, long)):
            if value>=100 and value<=900:
                st = _RESPONSE_STATUSES.get(value, '')
                if st:
                    self._status = '%d %s' % (value, st)
                else:
                    self._status = str(value)
            else:
                raise ValueError('bad response status code %d' % value)
        elif isinstance(value, basestring):
            if isinstance(value, unicode):
                self._status = value.encode('utf-8')
            if _RE_RESPONSE_STATUS.match(value):
                self._status = value
            else:
                raise ValueError('bad response status code %s' % value)
        else:
            raise TypeError('Bad type of response code.')
```

### WSGI 入口

```python
class WSGIApplication(object):
    def __init__(self, document_root=None, **kw):
        pass

    # 返回WSGI处理函数
    def get_wsgi_application(self, debug=False):
        pass

        # WSGI入口处理函数
        def wsgi(env, start_response): # 入口函数get_wsgi_application方法返回这个函数，标准的WSGI app
            ctx.application = _application
            ctx.request = Request(env) # 生成reuqest字典
            response = ctx.response = Response() # 生成response header对象
            try:
                r = fn_exec() # 生成response body
                if isinstance(r, Template): # 判断是否为Tmplate
                    r = self._temlate_engine(r.template_name, r.model) # 模版引擎渲染
                elif isinstance(r, unicode):
                    r = r.encode('utf-8')
                if r is None:
                    r = []
                start_response(response.status, response.headers) # 输出response body
                return r
            except RedirectError, e: # 如果为重定向
                response.set_header('Location', e.location)
                start_response(e.status, response.headers)
                return []
            except HttpError, e: # 如果为HTTP错误
                start_response(e.status, response.headers)
                if e.status == '404 Not Found' and os.path.exists(r'templates/404.html'):
                    return self._temlate_engine('404.html', dict())
                return ['<html><body><h1>', e.status, '</h1></body></html>']
            except Exception, e: # 其它错误500，如果为debug模式，把错误信息输出
                logging.exception(e)
                if not debug:
                    start_response('500 Internal Server Error', [])
                    return ['<html><body><h1>500 Internal Server Error</h1></body></html>']
                exc_type, exc_value, exc_traceback = sys.exc_info()
                fp = StringIO()
                traceback.print_exception(exc_type, exc_value, exc_traceback, file=fp)
                stacks = fp.getvalue()
                fp.close()
                start_response('500 Internal Server Error', [])
                return [
                   r'''<html><body><h1>500 Internal Server Error</h1><div style="font-family:Monaco, Menlo, Consolas, 'Courier New', monospace;"><pre>''',
                    stacks.replace('<', '&lt;').replace('>', '&gt;'),
                    '</pre></div></body></html>']
            finally:
                del ctx.request # 完成后清理当前的request和response header
                del ctx.response
        return wsgi
```
这里看到了一个关键的函数用于生成response body，下面来看一下这个fn_exec

```python
_re_route = re.compile(r'(\:[a-zA-Z_]\w*)') # 匹配只要是有:的就认为是有参数

# 构建参数分离的正则表达式:abc/ 即名为abc的参数
def _build_regex(path):
    re_list = ['^']
    is_var = False
    for v in _re_route.split(path):
        if is_var:
            varname = v[1:]
            re_list.append('(?P<%s>[^\\/]+)' % varname)
        else:
            s = ''
            for c in v:
                if c>='0' and c<='9':
                    s = s + c
                elif c>='a' and c<='z':
                    s = s + c
                elif c>='A' and c<='Z':
                    s = s + c
                else:
                    s = s + '\\' + c # 加\\是转移为\在r'\-'与'\\-'等同，在正则表达式下r'\-'等同'-'
            re_list.append(s)
        is_var = not is_var
    re_list.append('$')
    return ''.join(re_list)

# 定制url路由
class Route(object):
    '''
    区分函数是否为静态，不带参数，如果带参数则生成正则表达式
    '''
    def __init__(self, func):
        self.path = func.__route__
        self.method = func.__method__
        self.is_static = _re_route.search(self.path) is None # 判断path是否有参数
        if not self.is_static:
            self.route = re.compile(_build_regex(self.path)) # 构造参数匹配正则表达式
        self.func = func

    def match(self, path):
        m = self.route.match(path)
        if m:
            return m.groups()
        return None

    def __call__(self, *args):
        return self.func(*args)


class WSGIApplication(object):
    def __init__(self, document_root=None, **kw):
        '''
        定义了5个列表分别用力存储get post处理函数和拦截器
        '''
        self._running = False

        self._document_root = document_root

        self._get_static = {}
        self._post_static = {}

        self._get_dynamic = []
        self._post_dynamic = []

        self._interceptors = []

    def _check_not_running(self):
        if self._running:
            raise RuntimeError("can't modify WSGIApplication when it is running")

    # 添加一个URL定义
    def add_url(self, func): # 判断函数get post，动态添加到对应的列表
        self._check_not_running()
        route = Route(func)
        if route.is_static:
            if route.method == 'GET':
                self._get_static[route.path] = route
            elif route.method == 'POST':
                self._post_static[route.path] = route
        else:
            if route.method == 'GET':
                self._get_dynamic.append(route)
            elif route.method == 'POST':
                self._post_dynamic.append(route)
        logging.info('add route %s ' % route.method+':'+route.path)

    # 从模块添加url
    def add_model(self, mod):
        self._check_not_running()
        m = mod
        logging.info('add model %s' % mod.__name__)
        for name in dir(m):
            fn = getattr(m, name)
            if hasattr(fn, '__route__') and hasattr(fn, '__method__'):
                self.add_url(fn)

    # 添加一个Interceptor定义
    def add_interceptor(self, func):
        self._check_not_running()
        self._interceptors.append(func)
        logging.info('add interceptor %s ' % func.func_name)

    # 设置TemplateEngine
    @property
    def template_engine(self):
        self._template_engine

    @template_engine.setter
    def template_engine(self, engine):
        self._temlate_engine = engine

    # 返回WSGI处理函数
    def get_wsgi_application(self, debug=False):
        self._check_not_running()
        if debug:
            self._get_dynamic.append(StaticFileRoute())
        self._running = True

        _application = dict(document_root=self._document_root) # 这里只在取静态文件时有用，传入一个目录

        # 根据method与path路由获取处理函数，分为一般的和带参数的
        def fn_route(): # 获取path_info，根据path与method匹配处理函数
            path = ctx.request.path_info
            method = ctx.request.method
            if method == 'GET':
                fn = self._get_static.get(path, None)
                if fn:
                    return fn()
                for fn in self._get_dynamic:
                    args = fn.match(path) # 正则匹配出函数的参数
                    if args:
                        return fn(*args)
                raise notfound()
            if method == 'POST':
                fn = self._post_static.get(path, None)
                if fn:
                    return fn()
                for fn in self._post_dynamic:
                    args = fn.match(path)
                    if args:
                        return fn(*args)
                raise notfound()
            return badrequest()

        # 对于处理函数包裹上拦截器规则
        fn_exec = _build_interceptor_chain(fn_route, *self._interceptors) # 先不要看这里，这里用来为fn_route函数包裹上拦截器

        # WSGI入口处理函数
        def wsgi(env, start_response):
            pass
```
