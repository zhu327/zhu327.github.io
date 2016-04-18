---
date: 2015-03-14T16:17:35+08:00
title: RESTful API 理解
tags: ["web"]
---

### 参考

[理解RESTful架构](http://www.ruanyifeng.com/blog/2011/09/restful.html)  
[RESTful API 设计指南](http://www.ruanyifeng.com/blog/2014/05/restful_api.html)  
[来自HeroKu的HTTP API 设计指南(中文版)](http://get.jobdeer.com/343.get)

### 概念

REST == Representational State Transfer，意思是表现层状态转化，我们来分解这个概念。  
从字面以上理解这是一个操作的过程，实际上它省略了主语资源(Resources)，合起来就是资源的表现层状态转化。
<!--more-->

### 资源 Resources

需要操作的对象，可以是文本，图片，等等互联网上的各种资源，可以通过URI(统一资源定位符)来访问。

### 表现层 Representational

资源是客户端从服务器请求过去的，表现层代表了资源在传递过程中的表现形式，比如一个文本，可以是html，xml，json等等的形式，或者干脆是二机制。

表现层在Request HTTP header中通过`Accept`字段表示请求的数据类型，在Response HTTP header中通过`Content-Type`指定数据格式。

### 状态转化 State Transfer

在HTTP请求过程中对资源的操作过程就是状态的转化，使用HTTP的各种方法来实现对资源的操作。

1. GET  获取资源
2. POST 新增或者修改资源
3. PUT  修改资源
4. DELETE 删除资源

符合REST原则的API构建就是RESTful构架，从以上分析可以得出结论。

1. 每个URI代表一个资源
2. 客户端与服务端通过某种表现形式来传递资源
3. 通过HTTP方法来操作资源实现资源的状态转化

### 计划

有两个计划待实现：

1. [Django forum](https://github.com/zhu327/forum) RESTful API设计实现；
2. 微信公众平台Python开发框架设计实现。  
准备参考参考文章中的后2篇实现Django forum的RESTful API。
