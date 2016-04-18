---
date: 2015-07-02T16:37:48+08:00
title: openshift平台DIY环境配置Tornado
tags: ["python"]
---

> openshift官方和社区提供不少配置好的环境，也包括python2.7和python3.3下的tornado套件。  
一方面希望拥有更高的使用自由，另一方面希望熟悉一下openshift的DIY环境，本着这个目的，自行创建一个DIY环境，在上面安装运行python 2.7.10和Tornado 3.2。

【Step1、创建DIY环境】


【Step2、安装python 2.7.10】

通过`python -V`看到系统已预装`python 2.6.6`，直接在`$OPENSHIFT_DATA_DIR`下安装2.7.10可以正常使用，但需要注意其调用路径为`$OPENSHIFT_DATA_DIR/bin/python`。

<!--more-->
```shell
cd $OPENSHIFT_REPO_DIR
wget https://www.python.org/ftp/python/2.7.10/Python-2.7.10.tgz
tar -xzf Python-2.7.10.tgz
cd Python-2.7.10
./configure --prefix=$OPENSHIFT_DATA_DIR
make install
$OPENSHIFT_DATA_DIR/bin/python -V
```

看到Python 2.7.10说明安装成功。

【Step3、安装pip】

```shell
cd $OPENSHIFT_TMP_DIR
wget https://raw.github.com/pypa/pip/master/contrib/get-pip.py
$OPENSHIFT_DATA_DIR/bin/python get-pip.py
```

【Step4、安装Tornado 3.4】

```shell
$OPENSHIFT_DATA_DIR/bin/pip install tornado
```

通过`$OPENSHIFT_DATA_DIR/bin/pip freeze`查看Tornado3.4是否已经包含在已安装列表中

【Step5、修改action_hooks文件，这些文件定义应用的启动和终止】

```shell
cd $OPENSHIFT_REPO_DIR/.openshift/action_hooks
```

打开start文件vim start，注释掉原来的全部内容，并在尾部添加：

```shell
nohup $OPENSHIFT_DATA_DIR/bin/python $OPENSHIFT_REPO_DIR/diy/start.py > $OPENSHIFT_DIY_LOG_DIR/tornado_server.log 2>&1 &
```

打开stop文件vim stop，注释掉原来的全部内容，并在尾部添加：

```shell
source $OPENSHIFT_CARTRIDGE_SDK_BASH

if [ -z "$(ps -ef | grep start.py | grep -v grep)" ]
    then
    client_result "Application is already stopped"
else
    kill `ps -ef | grep start.py | grep -v grep | awk '{ print $2 }'` > /dev/null 2>&1
fi
```

【Step6、第一个Tornado程序】

```shell
cd $OPENSHIFT_REPO_DIR/diy/
rm *
vim start.py
```

写入：

```python
import os
import tornado.ioloop
import tornado.web

class MainHandler(tornado.web.RequestHandler):
    def get(self):
        self.write("Hello, world")

application = tornado.web.Application([
    (r"/", MainHandler),
])

if __name__ == "__main__":
    ip = os.environ['OPENSHIFT_DIY_IP']
    port = int(os.environ['OPENSHIFT_DIY_PORT'])
    application.listen(port, ip)
    tornado.ioloop.IOLoop.instance().start()
```

【Step7、启动Appliction】

```shell
ctl_all stop
ctl_all start
```

如果操作无误的话，hello world可以访问了。
