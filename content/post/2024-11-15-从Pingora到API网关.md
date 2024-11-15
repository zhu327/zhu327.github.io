---
title: "从Pingora到API网关：Rust实战"
date: 2024-11-15T10:55:52+08:00
draft: false
tags: ["Rust"]
---

### 前言

在学习Rust的过程中，我主要进行了一些小工具的练习，对Rust的内存安全性和性能优势有了初步的体会，但始终没有实现过一个完整的大型项目。最近随着Rust在高性能计算领域的应用不断拓展，尤其是Pingora等项目的发布，让我看到了Rust在网络通信领域中的潜力，也激发了我用Rust来实现一个API网关的兴趣。

在加入腾讯之前，我曾使用OpenResty和Kong进行API网关的开发工作，后来在腾讯进一步深入参与了APISix的云原生网关项目，逐步积累了关于API网关设计、实现以及性能调优方面的经验。API网关是一个兼具架构复杂度与性能要求的系统，涉及请求路由、流量控制、身份验证等多个关键模块，恰好可以充分发挥Rust的优势。因此，我决定以Rust为基础开发一个API网关，这不仅是为了提升自己的Rust技术，更希望通过此项目进一步巩固和提升在API网关方面的知识。

在具体实现中，我计划基于Pingora的设计思路，打造一个APISix网关的子集功能。我已在GitHub上发布了该项目的初始版本 <https://github.com/zhu327/pingsix>，目前实现了基础的standalone模式，包括基本的路由与反向代理功能。接下来，我将扩展插件定义，逐步实现一些典型的功能插件，最终目标是支持基于etcd的动态配置加载，以便适应多场景的API网关需求。

<!--more-->

### 实现：配置

在设计 pingsix 的配置文件时，我参考了 APISix 和 Pingora 的配置文件结构，结合自身项目目标和功能，简化了部分复杂内容，同时保持了核心功能的灵活性。配置文件包含以下三个关键部分：**基础配置**、**Listener 配置** 和 **资源配置**，以下是详细说明与完整 YAML 示例。

#### 1. 基础配置

基础配置部分直接沿用了 Pingora 的设计，用于定义网关的全局行为和运行参数。以下是完整配置示例：

```yaml
# pingora config example from https://github.com/cloudflare/pingora/blob/main/docs/user_guide/conf.md
pingora:
  version: 1              # 配置文件版本
  threads: 2              # 工作线程数量
  pid_file: /run/pingora.pid # 保存进程ID的文件路径
  upgrade_sock: /tmp/pingora_upgrade.sock # 热升级用的Unix socket文件
  user: nobody            # 运行网关的用户
  group: webusers         # 运行网关的用户组
```

**设计思路**：这一部分配置是网关运行时的基础，保持简洁的同时，与 Pingora 配置完全兼容，降低了配置的学习成本。

---

#### 2. Listener 配置

Listener 是 API 网关的入口，用于定义网关监听的地址、端口以及协议类型（HTTP/HTTPS）。以下是完整配置示例：

```yaml
listeners:
  - address: 0.0.0.0:8080 # HTTP监听地址和端口
  # - address: "[::1]:443"  # 示例：IPv6地址监听
  #   tls:                  # 配置HTTPS
  #     cert_path: /etc/ssl/server.crt # SSL证书路径
  #     key_path: /etc/ssl/server.key  # SSL私钥路径
  #   offer_h2: true         # 是否支持HTTP/2
```

**设计思路**：
- 当前支持 HTTP 和 HTTPS 协议，通过配置 `tls` 参数来开启 HTTPS。 未来可能加入类似 APISix 的 `Stream Proxy` 功能，用于支持 TCP/UDP 流量代理。

---

#### 3. 资源配置

资源配置是网关的核心部分，主要定义路由规则（Router）和后端服务（Upstream）的负载均衡策略。目前 pingsix 支持的功能子集以实现反向代理为主，以下是完整示例：

```yaml
# 参考APISix的Standalone模式配置示例：https://apisix.apache.org/zh/docs/apisix/3.1/stand-alone/
# 完整的Router，Upstream配置参考APISix的API文档：https://apisix.apache.org/zh/docs/apisix/admin-api/#route
routers:
  - id: 1                     # 路由唯一标识
    uri: /                    # 路由匹配的URI
    host: www.baidu.com       # 匹配的主机
    # 可选项：
    # uris: ["/","/test"]     # 支持多个URI匹配
    # hosts: ["www.baidu.com","www.taobao.com"] # 支持多个Host匹配
    # methods: ["GET", "POST"] # 限制请求方法
    # timeout:
    #   connect: 2           # 连接超时时间（秒）
    #   send: 3              # 发送超时时间（秒）
    #   read: 5              # 读取超时时间（秒）
    # priority: 10           # 路由优先级（数字越大，优先级越高）
    upstream:                # 定义后端服务
      nodes:
        "www.baidu.com": 1   # 后端服务地址及权重
      type: roundrobin       # 负载均衡类型（支持roundrobin、random等）
      checks:                # 健康检查配置
        active:              # 主动健康检查
          type: https        # 检查类型（http/https）
          timeout: 1         # 健康检查超时时间（秒）
          host: www.baidu.com # 检查目标主机
          http_path: /       # 健康检查路径
          https_verify_certificate: true # 是否验证HTTPS证书
          req_headers: ["User-Agent: curl/7.29.0"] # 请求头
          healthy:           # 健康服务的标准
            interval: 5      # 健康检查间隔时间（秒）
            http_statuses: [200, 201] # 判断健康的HTTP状态码
            successes: 2     # 连续成功次数
          unhealthy:         # 不健康服务的标准
            http_failures: 5 # 连续失败的HTTP请求次数
            tcp_failures: 2  # 连续失败的TCP请求次数
      pass_host: rewrite     # 是否透传主机信息
      upstream_host: www.baidu.com # 传递给后端的Host
      scheme: https          # 使用的协议（http/https）
```

**说明**：
- **Router** 定义了请求的路由规则，目前支持 `uri` 和 `host` 的匹配，还可以扩展支持 `methods`、`priority` 等参数。
- **Upstream** 配置支持负载均衡和健康检查，支持配置各种复杂均衡算法，还支持主动健康检查。

可以在 [GitHub](https://github.com/zhu327/pingsix/blob/main/config.yaml) 查看完整的配置文件。

---

#### 4. 配置加载

在 pingsix 中，配置加载的实现方式与 Pingora 完全兼容，主要通过复用 Pingora 的命令行解析和配置加载代码来完成。这些代码大部分是直接借鉴了 Pingora 的实现，并根据 pingsix 的需求做了一些简单的调整和优化。

##### 主要改进：
1. **使用 `serde` 进行配置解析**：配置文件仍然采用 YAML 格式，使用 `serde_yaml` 库进行解析。
2. **配置校验**：为了确保配置的正确性，我们引入了 `validator` crate，对配置项进行简单的校验，确保它们符合预期。
3. **默认值和拷贝**：通过 `serde` 的 `default` 特性简化了默认值的处理，使用 `Clone` derive 特性实现了配置的拷贝。

##### 配置加载实现

配置的加载流程和 Pingora 基本一致，代码结构也与 Pingora 相似。我们首先读取 YAML 配置文件，然后通过 `serde_yaml` 进行解析，最后使用 `validator` 校验配置项的合法性。

具体的实现可以参考 [pingsix 的配置加载代码](https://github.com/zhu327/pingsix/blob/main/src/config/mod.rs)，该文件展示了配置解析、校验和默认值处理的实现方式。

### 实现：上游

从 APISix 的上游配置功能可以看出，网关的上游模块需要具备以下核心功能：

1. **DNS 解析**：支持上游服务的域名解析。
2. **负载均衡算法**：支持多种负载均衡算法和自定义 hash key。
3. **健康检查**：实现主动健康检查，确保上游服务的可用性。
4. **超时与重试配置**：为连接、发送和读取设置超时时间，并在请求失败时进行重试。
5. **协议支持**：支持 HTTP 和 HTTPS 上游协议。

#### 上游模块实现结构

根据 Pingora 的抽象设计，我们将上游处理拆分为两个主要模块，分别负责服务发现和负载均衡：

1. **服务发现模块**  
   服务发现模块位于 `discovery.rs`，支持静态 IP 配置、域名 DNS 解析，并允许 HTTP/HTTPS 混合配置。详情见 [discovery.rs 源码](https://github.com/zhu327/pingsix/blob/main/src/proxy/discovery.rs)。

2. **负载均衡模块**  
   负载均衡模块位于 `lb.rs`，支持 roundrobin、random、fnvhash 和 ketama 等算法，并按照 APISix 实现了部分基于 `vars`、`header` 和 `cookie` 的 hash 计算。负载均衡模块可扩展不同的 hash key 类型。更多实现细节见 [lb.rs 源码](https://github.com/zhu327/pingsix/blob/main/src/proxy/lb.rs)。

#### 实现心得

1. **多态设计**：通过 `enum` 实现负载均衡算法的多态性，将不同算法的差异封装在模块内，便于调用和扩展。
   
2. **配置转换简化**：使用 `From` trait 将配置结构转换为业务对象，简化代码，提高可读性。

3. **健康检查集成**：Pingora 的健康检查逻辑与上游绑定在一起，而健康检查后台服务需要在 server 启动时注入。因此，我们在上层结构中将上游和健康检查逻辑绑定，通过 `Option` 灵活控制其生命周期。

以上实现方法确保了代码结构的简洁性和模块的可复用性，建议阅读相关源码进一步了解实现细节。

### 实现：路由

路由模块参考了 [APISix 的 radixtree_host_uri 路由方式](https://github.com/apache/apisix/tree/master/apisix/http/router)，通过 `host` 和 `uri` 的组合实现高效的路由匹配。

在 pingsix 中，我们使用了 Rust 的 `matchit` crate，这是一个高性能的 Radix Tree 路由库，用来实现与 APISix 相同的功能。`matchit` 提供了灵活的路由匹配能力，支持精确匹配和通配符规则。

此外，路由逻辑还结合了负载均衡（lb）模块的方法，实现请求的精准路由，并支持通过路由的 `timeout` 配置覆盖默认的上游超时设置。

完整实现可以参考 [router.rs 源码](https://github.com/zhu327/pingsix/blob/main/src/proxy/router.rs)。

### 实现：Pingora Service 与 Server

#### Service 实现

Pingora 的 `Service` 概念用于定义 HTTP 请求处理的各个阶段，其处理流程与 Nginx 的处理阶段类似（参考 [Pingora 的阶段设计](https://github.com/cloudflare/pingora/blob/main/docs/user_guide/phase.md)）。在 pingsix 中，通过实现 `ProxyService` trait，可以构建一个完整的反向代理服务，并在以下阶段插入自定义逻辑：

- **request**: 请求接收后可进行修改或校验。  
- **upstream_request**: 转发到上游前处理请求。  
- **upstream_response**: 接收上游响应后进行处理。  
- **response**: 返回客户端前对响应进行最终修改。

这种设计让我们可以通过插件灵活扩展功能，例如添加认证、流量控制等。核心代码实现可参考 [pingsix proxy 模块](https://github.com/cloudflare/pingora/blob/main/src/mod.rs)。

#### Server 实现

Server 层负责协调配置、路由和服务启动等核心功能：

1. **加载配置与路由**  
   从配置文件中加载路由规则和监听地址，并初始化所需的服务模块。

2. **启动监听器**  
   支持多协议监听（如 HTTP/HTTPS），并动态绑定配置的监听端口。

3. **Service 扩展**  
   在 Server 中可以注册多种 `Service` 实现：  
   - `ProxyService`: 用于处理反向代理逻辑。  
   - `BackgroundService`: 如健康检查后台任务。  
   - 自定义扩展 Service，例如自动申请证书等。

通过这种设计，Server 可以灵活地管理多个服务实例，扩展功能变得简单且直观。完整的实现代码可参考 [pingsix 的 main.rs](https://github.com/zhu327/pingsix/blob/main/src/main.rs)。

### 规划

接下来，我们将基于当前的反向代理功能进一步完善 API 网关的核心扩展能力，规划包括：

1. **插件支持**  
   引入插件机制，支持动态加载和执行，逐步实现 API 网关的关键插件（如认证、限流、日志等），提升功能的灵活性与可扩展性。

2. **动态配置管理**  
   集成 etcd，实现配置的动态加载和更新，以便在无需重启服务的情况下动态调整路由、上游配置等。

3. **上游与服务的独立配置**  
   提供上游（upstream）和服务（service）的单独配置抽象，使得复杂业务需求下的配置更清晰、模块化。

---

经过以上设计，我们已经构建了一个完整的基础反向代理功能，但要实现 API 网关的核心价值，接下来的重点将聚焦在插件系统的设计与实现，以便支持更丰富的功能扩展。

### 参考

- [APISix](https://github.com/apache/apisix) - 开源、高性能的 API 网关，提供丰富的插件支持和灵活的配置管理。
- [Pingora](https://github.com/cloudflare/pingora) - Cloudflare 开发的高性能反向代理，采用 Rust 实现，设计灵感来源于 Nginx。

其他相关项目：

- [pingap](https://github.com/vicanso/pingap) - 一个使用 Rust 实现的轻量级 API 网关，适合参考其架构和插件设计。
- [river](https://github.com/memorysafety/river) - Rust 实现的分布式流量管理系统，支持动态路由和负载均衡功能。
- [penguin](https://github.com/caibirdme/penguin) - Rust 编写的反向代理服务器，用于学习和参考其路由与负载均衡实现。
