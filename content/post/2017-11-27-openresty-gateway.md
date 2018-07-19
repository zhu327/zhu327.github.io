---
date: 2017-11-27T18:20:31+08:00
title: OpenResty与API Gateway
---

公司业务向微服务方向迁移实践中, API Gateway成为接入层最重要的部分, 在完成开发的同时给同事做了一次OpenResty的分享, 以下是分享的内容整理.

---

### 当我谈论高性能时我们谈论什么 ?

讨论时间, 大家自由发挥

- 机器
- 语言
- 架构
  - reactor
  - coroutine
  - cache

<!--more-->

### 学习指南 #1

- reactor
  - <http://blog.csdn.net/u013074465/article/details/46276967>
- coroutine
  - <http://blog.csdn.net/wyx819/article/details/45420017>
- Tornado代码阅读笔记 IOLoop
  - <http://t.cn/R5XxpMM>

---

### 同步编程

- 多线程, 用户态/内核态相互切换
- 阻塞, 带来线程切换

### 异步编程

- 事件驱动
- 回调
- 协程

---

## OpenResty

- Nginx
- `lua-nginx-module`
- lua resty libs
  - <https://github.com/iresty/nginx-lua-module-zh-wiki#description>

*无聊的分割线*

- 基于lua的Nginx开发
- Nginx的异步机制+lua coroutine
- 够用的内置lua libs cosocket异步支持
- 内置cache

*ngx_lua模块的原理*

* 每个worker（工作进程）创建一个Lua VM，worker内所有协程共享VM；
* 将Nginx I/O原语封装后注入 Lua VM，允许Lua代码直接访问；
* 每个外部请求都由一个Lua协程处理，协程之间数据隔离；
* Lua代码调用I/O操作等异步接口时，会挂起当前协程（并保护上下文数据），而不阻塞worker；
* I/O等异步操作完成时还原相关协程上下文数据，并继续运行；

---

### 学习指南 #2

- OpenResty 系列课程
  - <http://www.stuq.org/course/1015>
- agentzh 的 Nginx 教程
  - <https://openresty.org/download/agentzh-nginx-tutorials-zhcn.html>
- OpenResty 最佳实践
  - <https://moonbingbing.gitbooks.io/openresty-best-practices/content/>
- nginx-lua-module-zh-wiki
  - <https://github.com/iresty/nginx-lua-module-zh-wiki>

---

### API Gateway

- 微服务
- 入口
- 授权、监控、负载均衡、缓存、请求分片和管理、静态响应处理


### 学习指南 #3

- 微服务
  - <http://blog.csdn.net/wurenhai/article/details/37659335/>
- API Gateway
  - <http://dockone.io/article/482>

<img src="https://camo.githubusercontent.com/0e1bbef1559f33dd31b9f352c83d2caf2bc4e61b/687474703a2f2f636c2e6c792f696d6167652f3142334a33623368314831632f496d616765253230323031352d30372d30372532306174253230362e35372e3235253230504d2e706e67" width="720">

---

### Kong

- 基于Openresty
- 自带武器库
- 灵活的插件定制

### 学习指南 #4

- Kong源码分析
  - <http://cyukang.com/2017/07/02/kong-intro.html>
- 深入理解orange
  - <https://github.com/linger1216/understanding-orange>

---

### 定制需求

- eebo auth
  - 独立auth模块
  - 灵活配置api是否需要auth
  - Python服务无需关注认证细节
- eebo limiting
  - 基于内置rate-limiting开发
  - 提供`company_id/user_id`维度
- eebo balancer
  - 提供私有服务
  - 强大的分布式分发

### 自带的插件

- CORS
  - 用于处理跨域请求
- IP Restriction
  - IP 黑白名单
- Correlation ID
  - 为每个请求生成唯一请求id
  - 方便定位, 跟踪日志
- Syslog
  - 推送日志到Syslog-