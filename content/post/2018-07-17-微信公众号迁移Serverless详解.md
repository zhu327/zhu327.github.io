---
title: "微信公众号迁移Serverless详解"
date: 2018-07-17T00:12:12+08:00
draft: false
---

3月腾讯云函数计算开放测试, 看到的第一反应是这种Serverless太适合做微信公众号的后端来实现自动应答了, 尝试把我服务了3年的一个公众号迁移到腾讯云函数计算, 结果因为API gateway的一个功能缺失搁置了, 这周腾讯云API gateway终于补上了集成响应的能力, 能正常服务我的公众号, 这里记录下实现过程.

<!--more-->

### 改造werobot

<https://github.com/zhu327/ifwechat>

ifwechat这个公众号是基于[werobot](https://github.com/offu/WeRoBot)实现的, 之前以WSGI的方式部署在[SAE](https://sae.sina.com.cn/)上, 而不同于WSGI, 函数计算从API gateway触发的事件是一个Python的字典, 需要从这个事件字典里面获取http request的信息来调用werobot的方法. API gateway的触发事件参考: <https://cloud.tencent.com/document/product/583/13197>

需要实现一个从http request事件到werobot的message的wrapper handler:

```python
# coding: utf-8

from handlers import robot

robot.config.from_pyfile('configs.py')


def werobot_handler(event):
    timestamp = event["queryString"].get("timestamp", "")
    nonce = event["queryString"].get("nonce", "")
    signature = event["queryString"].get("signature", "")

    if not robot.check_signature(timestamp=timestamp,
                                 nonce=nonce,
                                 signature=signature):
        return "error: wrong wechat signature"
    if event["requestContext"]["httpMethod"] == "GET":
        return event["queryString"].get("echostr", "")
    elif event["requestContext"]["httpMethod"] == "POST":
        body = event["body"].event["body"].replace('\\u003c', '<').replace('\\u003e', '>').decode('utf8')
        message = robot.parse_message(
            body,
            timestamp=timestamp,
            nonce=nonce,
            msg_signature=event["queryString"].get("msg_signature", ""))
        return robot.get_encrypted_reply(message)
```

在API gateway上创建API时需要勾选集成响应功能, 并且在函数入口返回的格式如下:

```python
# coding: utf-8

from scf_werobot import werobot_handler


def main_handler(event, context):
    if "requestContext" not in event.keys():
        return "error: event is not come from api gateway"

    content = werobot_handler(event)

    return {
        "statusCode": 200,
        "headers": {"Content-Type": "application/xml"},
        "body": content
    }
```

### 通过cos实现werobot的session

除了计算外, ifwechat还用到了werobot的session来存微信用户与ifttt用户之间的对应关系, 在SAE上部署session数据是存在SAE的kvdb里的, 但是在腾讯云这里就没有免费的Redis或者MySQL来用了. 在研究zappa这个serverless框架的时候, 发现他们用AWS S3实现了一个NoDB的库可用用来做kvdb, 而腾讯云对标S3存储的就是cos, 把NoDB fork修改S3代码改成cos sdk调用, 就有了这个[NoDB for Tencent Cloud COS](https://github.com/zhu327/NoDB)

继承werobot的SessionStorage实现用NoDB来做session backend:

```python
# coding: utf-8

from werobot.session import SessionStorage
from nodb import NoDB


class NoDBStorage(SessionStorage):
    def __init__(self, secret_id, secret_key, region, bucket):
        self.nodb = NoDB(secret_id, secret_key, region)
        self.nodb.bucket = bucket

    def get(self, id):
        return self.nodb.load(id) or {}

    def set(self, id, value):
        self.nodb.save(value, id)

    def delete(self, id):
        self.nodb.delete(id)

    def __getitem__(self, id):
        return self.get(id)

    def __setitem__(self, id, session):
        self.set(id, session)

    def __delitem__(self, id):
        self.delete(id)
```

打包所有代码为zip文件, 并发布到scf就完成了这个迁移过程.

### 关于zappa

从迁移过程的体验来看, 功能的开发还是很简单的, 只是部署的过程不是很友好, 如果能有一个类似于zappa这样的自动化部署框架来对接到腾讯云函数计算, 相信对开发者来说会更友好.

