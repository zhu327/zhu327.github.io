---
title: "pingsix-ingress-controller启动"
date: 2025-10-20T10:00:52+08:00
draft: false
---

> <https://github.com/zhu327/pingsix>

PingSIX 自从上次重构之后, 我一直在考虑如何扩展这个项目的功能与使用场景, 有2个方向可以考虑:

1. 支持proxy-wasm插件
2. 实现pingsix-ingress-controller

在调研了proxy-wasm的相关功能后, 结论是由于pingora面向的场景是CDN的反向代理, 所以没有考虑过方便的修改请求体与响应体, 这就造成很难基于pingora来实现proxy-wasm的ABI, 如果我要自己定义一个wasm的接口协议, 没法复用社区现有的proxy-wasm插件, 那就没必要了, 不如直接写Rust的插件. 相关参考内容:

- [proxy-wasm support](https://github.com/cloudflare/pingora/issues/17)
- [pingora 能做什么和不能做什么](https://segmentfault.com/a/1190000045232953)

在放弃了proxy-wasm的支持后, 我开始调研如何实现pingsix-ingress-controller, 由于pingsix的的资源定义是参考apisix来实现的, 所以就直接参考[apisix-ingress-controller](https://github.com/apache/apisix-ingress-controller)来实现我们自己的[pingsix-ingress-controller](https://github.com/zhu327/pingsix-ingress-controller).

<!--more-->

### 1. apisix-ingress-controller架构

<img src="https://github.com/user-attachments/assets/1ecf116a-378f-4357-a597-5bafb56991fd" alt="Image1" width="700px" />

#### 1. K8s Resources Watch Layer (资源监听层)
- 各种Controller通过Kubernetes的Watch机制监听对应的资源变化
- 支持的资源类型包括：
  - Gateway API: HTTPRoute, Gateway, GRPCRoute, TCPRoute, UDPRoute, TLSRoute
  - Kubernetes原生: Ingress, IngressClass
  - APISIX CRD: ApisixRoute, ApisixGlobalRule, ApisixTls, ApisixConsumer, ApisixUpstream
  - 自定义: Consumer, GatewayProxy

#### 2. Provider Layer (提供者层)
- Provider接收Controller的Update/Delete请求
- TranslateContext收集所有依赖资源（Services, Secrets, EndpointSlices等）
- 为Translator提供完整的上下文信息

#### 3. Translator Layer (翻译层)
- 将K8s资源翻译成ADC资源描述
- 每种资源类型都有对应的Translator方法
- 输出TranslateResult，包含：
  - Services (路由规则)
  - SSL/TLS (证书)
  - Consumers (消费者)
  - GlobalRules (全局规则)
  - PluginMetadata (插件元数据)

#### 4. ADC Client Layer (ADC客户端层)
- **ConfigManager**: 管理多个GatewayProxy的配置
- **Store/MemDB**: 内存数据库，存储ADC资源状态
- **StoreDelta**: 对比新旧配置，计算差异
- 将变更任务传递给Executor执行

#### 5. ADC Executor Interface (执行器接口)
ADC Executor提供统一的Execute接口，支持三种实现：
- **HTTPADCExecutor**: 通过HTTP调用ADC Server（推荐方式）
- **DefaultADCExecutor**: 通过命令行调用adc命令

#### 6. ADC HTTP Server (ADC HTTP服务器)
ADC HTTP Server的核心流程：
1. 接收`/sync`端点的PUT请求
2. 解析ADCServerRequest（包含opts和config）
3. 根据label-selector从APISIX拉取现有资源
4. 将ADC资源描述转换为APISIX资源格式
5. 对比已拉取的资源，计算差异（Diff）
6. 调用APISIX Admin API执行创建/更新/删除操作
7. 返回SyncResult（包含成功/失败状态）

#### 7. APISIX Data Plane (APISIX数据平面)
最终在APISIX中创建/更新/删除的资源：
- Routes (路由)
- Services (服务)
- Upstreams (上游)
- SSL/TLS (证书)
- Consumers (消费者)
- Global Rules (全局规则)
- Plugin Metadata (插件元数据)

#### 8. 核心特性

1. **多GatewayProxy支持**: 通过ConfigManager管理多个APISIX实例的配置
2. **Label Selector**: 支持通过标签选择器过滤资源
3. **增量同步**: 通过MemDB对比差异，只同步变更的资源
4. **错误处理**: 完善的错误收集和状态更新机制
5. **灵活的执行方式**: 支持HTTP、命令行多种执行模式
6. **资源隔离**: 通过label实现不同资源的隔离和管理

### 2. apisix-ingress-controller的经验

虽然以前也写过一些operator的代码, 但是在学习apisix-ingress-controller代码的过程中, 我还是学到了一些新的东西, 在controller watch一类资源的时候, 我们可以watch所有关联的资源类型, 一旦这些关联的资源类型有变更, 就可以进入统一的变更流程, 这样就减少了我们写controller的逻辑复杂度, 所有的关联资源的变更都会触发主资源的更新.

```golang
// SetupWithManager sets up the controller with the Manager.
func (r *IngressReconciler) SetupWithManager(mgr ctrl.Manager) error {
	r.genericEvent = make(chan event.GenericEvent, 100)

	return ctrl.NewControllerManagedBy(mgr).
		For(&networkingv1.Ingress{},
			builder.WithPredicates(
				MatchesIngressClassPredicate(r.Client, r.Log),
			),
		).
		WithEventFilter(
			predicate.Or(
				predicate.GenerationChangedPredicate{},
				predicate.AnnotationChangedPredicate{},
				predicate.NewPredicateFuncs(TypePredicate[*corev1.Secret]()),
			),
		).
		Watches(
			&networkingv1.IngressClass{},
			handler.EnqueueRequestsFromMapFunc(r.listIngressForIngressClass),
			builder.WithPredicates(
				predicate.NewPredicateFuncs(r.matchesIngressController),
			),
		).
		Watches(
			&discoveryv1.EndpointSlice{},
			handler.EnqueueRequestsFromMapFunc(r.listIngressesByService),
		).
		Watches(
			&corev1.Secret{},
			handler.EnqueueRequestsFromMapFunc(r.listIngressesBySecret),
		).
		Watches(&v1alpha1.BackendTrafficPolicy{},
			handler.EnqueueRequestsFromMapFunc(r.listIngressForBackendTrafficPolicy),
			builder.WithPredicates(
				BackendTrafficPolicyPredicateFunc(r.genericEvent),
			),
		).
		Watches(&v1alpha1.HTTPRoutePolicy{},
			handler.EnqueueRequestsFromMapFunc(r.listIngressesByHTTPRoutePolicy),
			builder.WithPredicates(httpRoutePolicyPredicateFuncs(r.genericEvent)),
		).
		Watches(&v1alpha1.GatewayProxy{},
			handler.EnqueueRequestsFromMapFunc(r.listIngressesForGatewayProxy),
		).
		WatchesRawSource(
			source.Channel(
				r.genericEvent,
				handler.EnqueueRequestsFromMapFunc(r.listIngressForGenericEvent),
			),
		).
		Complete(r)
}
```

IngressReconciler Watch 事件详解

根据代码分析，`IngressReconciler` 在 `SetupWithManager` 方法中配置了多个 watch 事件。让我为您详细解释每个事件的作用和逻辑：

#### 1. **主资源 Watch - Ingress** (第70-81行)

```go
For(&networkingv1.Ingress{}, builder.WithPredicates(
    MatchesIngressClassPredicate(r.Client, r.Log),
))
```

**作用**：监听 Ingress 资源本身的变化

**Predicates（过滤条件）**：
- `MatchesIngressClassPredicate`: 只处理由当前控制器管理的 IngressClass 的 Ingress
- `GenerationChangedPredicate`: 资源的 Generation 发生变化（spec 修改）
- `AnnotationChangedPredicate`: 注解发生变化
- `TypePredicate[*corev1.Secret]()`: 用于 Secret 类型判断

**逻辑**：这是主要的监听对象，当 Ingress 的规格或注解变化时触发 Reconcile


#### 2. **IngressClass Watch** (第82-88行)

```go
Watches(
    &networkingv1.IngressClass{},
    handler.EnqueueRequestsFromMapFunc(r.listIngressForIngressClass),
    builder.WithPredicates(
        predicate.NewPredicateFuncs(r.matchesIngressController),
    ),
)
```

**作用**：监听 IngressClass 资源的变化

**触发条件**：
- `matchesIngressController`: 只监听由当前控制器管理的 IngressClass（通过 `spec.controller` 字段匹配）

**逻辑** (`listIngressForIngressClass`)：
1. 检查 IngressClass 是否是默认类（通过注解 `ingressclass.kubernetes.io/is-default-class`）
2. 如果是默认类：列出所有未指定 IngressClassName 或指定为该类的 Ingress
3. 如果不是默认类：通过索引查找使用该 IngressClass 的所有 Ingress
4. 返回需要 reconcile 的 Ingress 列表

**使用场景**：当 IngressClass 的配置变化时，需要重新处理所有使用该类的 Ingress

#### 3. **EndpointSlice Watch** (第89-92行)

```go
Watches(
    &discoveryv1.EndpointSlice{},
    handler.EnqueueRequestsFromMapFunc(r.listIngressesByService),
)
```

**作用**：监听后端服务的 Endpoint 变化

**逻辑** (`listIngressesByService`)：
1. 从 EndpointSlice 的 label 中提取 Service 名称（`discovery.k8s.io/service-name`）
2. 通过索引 `ServiceIndexRef` 查找引用该 Service 的所有 Ingress
3. 过滤出由当前控制器管理的 Ingress
4. 返回需要 reconcile 的 Ingress 列表

**使用场景**：当后端 Pod 的 IP 地址变化（扩缩容、重启等）时，需要更新 APISIX 的 upstream 配置

#### 4. **Secret Watch** (第93-96行)

```go
Watches(
    &corev1.Secret{},
    handler.EnqueueRequestsFromMapFunc(r.listIngressesBySecret),
)
```

**作用**：监听 TLS 证书 Secret 的变化

**逻辑** (`listIngressesBySecret`)：
1. 通过索引 `SecretIndexRef` 查找直接引用该 Secret 的 Ingress（TLS 配置）
2. 查找引用该 Secret 的 GatewayProxy（用于 provider 认证）
3. 如果 GatewayProxy 引用了该 Secret，找到使用该 GatewayProxy 的 IngressClass
4. 再找到使用这些 IngressClass 的所有 Ingress
5. 去重后返回所有需要 reconcile 的 Ingress

**使用场景**：
- TLS 证书更新或轮换
- GatewayProxy 的 AdminKey Secret 变化

#### 5. **BackendTrafficPolicy Watch** (第97-102行)

```go
Watches(&v1alpha1.BackendTrafficPolicy{},
    handler.EnqueueRequestsFromMapFunc(r.listIngressForBackendTrafficPolicy),
    builder.WithPredicates(
        BackendTrafficPolicyPredicateFunc(r.genericEvent),
    ),
)
```

**作用**：监听后端流量策略的变化

**Predicates 逻辑**：
- **Create**: 返回 true，新建时触发
- **Delete**: 返回 true，删除时触发
- **Update**: 检测 `targetRefs` 的变化
  - 找出被移除的 targetRefs
  - 将包含被移除 targetRefs 的旧对象发送到 genericEvent channel
  - 这样可以清理不再被引用的资源

**逻辑** (`listIngressForBackendTrafficPolicy`)：
1. 遍历 Policy 的所有 `targetRefs`（引用的 Service）
2. 通过索引查找使用这些 Service 的 Ingress
3. 去重后返回需要 reconcile 的 Ingress 列表

**使用场景**：配置后端流量策略（如负载均衡算法、健康检查等）

#### 6. **HTTPRoutePolicy Watch** (第103-106行)

```go
Watches(&v1alpha1.HTTPRoutePolicy{},
    handler.EnqueueRequestsFromMapFunc(r.listIngressesByHTTPRoutePolicy),
    builder.WithPredicates(httpRoutePolicyPredicateFuncs(r.genericEvent)),
)
```

**作用**：监听 HTTP 路由策略的变化

**Predicates 逻辑**：
- **Create/Delete**: 返回 true
- **Update**: 检测 `targetRefs` 的变化
  - 找出被移除的 targetRefs
  - 将包含被移除 targetRefs 的旧对象发送到 genericEvent channel

**逻辑** (`listIngressesByHTTPRoutePolicy`)：
1. 遍历 Policy 的所有 `targetRefs`
2. 过滤出 Kind 为 `Ingress` 的引用
3. 获取这些 Ingress 对象
4. 返回需要 reconcile 的 Ingress 列表

**使用场景**：配置 HTTP 路由级别的策略（如重写、重定向、超时等）

#### 7. **GatewayProxy Watch** (第107-109行)

```go
Watches(&v1alpha1.GatewayProxy{},
    handler.EnqueueRequestsFromMapFunc(r.listIngressesForGatewayProxy),
)
```

**作用**：监听 GatewayProxy 配置的变化

**逻辑** (`listIngressesForGatewayProxy` -> `listIngressClassRequestsForGatewayProxy`)：
1. 通过索引 `IngressClassParametersRef` 查找引用该 GatewayProxy 的 IngressClass
2. 对每个 IngressClass，调用 `listIngressForIngressClass` 获取相关 Ingress
3. 去重后返回所有需要 reconcile 的 Ingress

**使用场景**：
- GatewayProxy 的 APISIX 地址变化
- 发布服务配置变化
- Provider 配置变化

#### 8. **Generic Event Channel** (第110-116行)

```go
WatchesRawSource(
    source.Channel(
        r.genericEvent,
        handler.EnqueueRequestsFromMapFunc(r.listIngressForGenericEvent),
    ),
)
```

**作用**：处理通过 channel 发送的自定义事件

**逻辑** (`listIngressForGenericEvent`)：
- 根据对象类型路由到相应的处理函数：
  - `BackendTrafficPolicy` -> `listIngressForBackendTrafficPolicy`
  - `HTTPRoutePolicy` -> `listIngressesByHTTPRoutePolicy`

**使用场景**：
- 处理 Policy 的 targetRefs 被移除时的清理工作
- 确保当资源不再被引用时，能正确更新相关配置

#### 整体工作流程

```
1. 事件触发 → 2. Predicate 过滤 → 3. MapFunc 映射 → 4. Reconcile 队列 → 5. Reconcile 执行
```

#### Reconcile 主要步骤：

1. **获取 Ingress 对象**：如果不存在则执行删除逻辑
2. **查找 IngressClass**：确定配置来源
3. **处理 IngressClass Parameters**：加载 GatewayProxy 配置
4. **处理 TLS**：加载证书 Secret
5. **处理 Backends**：加载 Service 和 EndpointSlice
6. **处理 HTTPRoutePolicy**：应用路由策略
7. **处理 BackendTrafficPolicy**：应用后端流量策略
8. **更新 APISIX 配置**：通过 Provider 同步到 APISIX
9. **更新状态**：更新 Ingress 和相关资源的状态

#### 关键设计特点

1. **索引优化**：使用 Field Indexer 快速查找资源关系
2. **级联更新**：依赖资源变化时自动触发主资源更新
3. **去重机制**：避免重复处理同一个 Ingress
4. **事件通道**：使用 genericEvent channel 处理复杂的清理场景
5. **条件过滤**：通过 Predicate 减少不必要的 Reconcile

#### indexer性能优化

可以看到在上面的watch逻辑中有很多的list操作, 比如listIngressForBackendTrafficPolicy, 这个时候就需要事先在k8s client的indexer中建立索引来优化查询速度, 避免list全扫数据:

```go
func setupIngressIndexer(mgr ctrl.Manager) error {
	// create IngressClass index
	if err := mgr.GetFieldIndexer().IndexField(
		context.Background(),
		&networkingv1.Ingress{},
		IngressClassRef,
		IngressClassRefIndexFunc,
	); err != nil {
		return err
	}

	// create Service index for quick lookup of Ingresses using specific services
	if err := mgr.GetFieldIndexer().IndexField(
		context.Background(),
		&networkingv1.Ingress{},
		ServiceIndexRef,
		IngressServiceIndexFunc,
	); err != nil {
		return err
	}

	// create secret index for TLS
	if err := mgr.GetFieldIndexer().IndexField(
		context.Background(),
		&networkingv1.Ingress{},
		SecretIndexRef,
		IngressSecretIndexFunc,
	); err != nil {
		return err
	}

	if err := mgr.GetFieldIndexer().IndexField(
		context.Background(),
		&networkingv1.Ingress{},
		TLSHostIndexRef,
		IngressTLSHostIndexFunc,
	); err != nil {
		return err
	}

	return nil
}
```

##### 📊 **索引概览表**

| 索引名称 | 索引字段 | 索引函数 | 主要用途 |
|---------|---------|---------|---------|
| IngressClassRef | `ingressClassRef` | `IngressClassRefIndexFunc` | 根据 IngressClass 查找 Ingress |
| ServiceIndexRef | `serviceRefs` | `IngressServiceIndexFunc` | 根据后端 Service 查找 Ingress |
| SecretIndexRef | `secretRefs` | `IngressSecretIndexFunc` | 根据 TLS Secret 查找 Ingress |
| TLSHostIndexRef | `tlsHostRefs` | `IngressTLSHostIndexFunc` | 根据 TLS 主机名查找 Ingress |

Ingress 的 4 个索引形成了一个完整的查询体系：

1. **IngressClassRef**：管理层面 - 按控制器分组
2. **ServiceIndexRef**：数据平面 - 后端服务关联
3. **SecretIndexRef**：安全层面 - TLS 证书管理
4. **TLSHostIndexRef**：域名层面 - SSL 配置管理

这些索引确保了当任何依赖资源变化时，控制器都能**快速、准确**地找到需要更新的 Ingress，实现**高效的级联更新**和**实时配置同步**。

### 3. pingsix-ingress-controller架构决策

从apisix-ingress-controller的架构中, 我们知道apisix抽象了一种ADC(API Declarative CLI)的资源类型用来在ingress与apisix资源之间作为中间的桥梁, 如果直接使用apisix-ingress-controller来对接pingsix的话, 就需要pingsix完整的实现apisix的admin api, 并且还需要在使用时启动etcd.

在我的印象中曾经看到过apisix-ingress-controller实现过一个不需要etcd的方案:

- [Embrace the Lightweight APISIX Ingress Controller Without etcd Dependency](https://apisix.apache.org/blog/2023/10/18/ingress-apisix/)

然后我在apisix-ingress-controller 1.8.x 版本的代码下找到了这个这个实现方式, 只是当前的 2.0.x 版本在引入了ADC相关的功能后去掉了[etcd-adapter](https://github.com/api7/etcd-adapter), 那我在考虑实现我的pingsix-ingress-controller时, 为了避免直接改动pingsix的代码, 并且也不希望在ingress启动时依赖etcd, 所以决定重新引入etcd-adapter, 然后为了后续pingsix-ingress-controller能同步apisix-ingress-controller的上游更新, 我决定在现有的ADC Executor的接口的基础上, 实现pingsix的Executor, 这样就可以在避免直接修改apisix的逻辑代码, 使用一种adapter的方式来实现我们自己的pingsix-ingress-controller.

<img src="https://github.com/user-attachments/assets/a10a7162-ea7d-4407-b7a6-b5481823592b" alt="Image2" width="700px" />

可以看到我们实现的这个Executor, 其实就是重复了一遍ADC http serve的逻辑, 但是最终资源的数据是写入到etcd-adapter中的, 我们实现了一个apisix-ingress-controller最底层的抽象, 通过这种低成本的改造我们后续还可以继续同步apisix-ingress-controller的变更, 并不会造成代码冲突.

### 总结

在实现[pingsix-ingress-controller](https://github.com/zhu327/pingsix-ingress-controller)的过程中, 我完整的阅读了apisix-ingress-controller的代码, 收获了一些写operator的技巧, 然后在分析现有apisix-ingress-controller的代码架构时, 决定通过扩展ADC Executor的实现来实践了面向对象的开闭原则, 对于后续的代码更新合并开了一个好头.