---
title: "从Pingora到API网关：总结"
date: 2024-12-19T10:55:52+08:00
draft: false
tags: ["Rust"]
---

### 前言

> <https://github.com/zhu327/pingisx>

经过一个多月的开发，终于完成了使用Rust实现一个API网关的目标，通过这个项目，我基本上把Pingora的核心功能都摸透了，然后也再次加深了对APISIX的理解。总的来说PingSIX基于Pingora实现了APISIX的核心功能，是APISIX功能的一个子集。

开发这个项目本身也是为了学习Rust的使用，在开发的过程中确实也遇到了很多的问题，在ChatGPT的帮助下，这些问题得到了解决，也让我对Rust有了更深刻的理解。下面我通过4个方面来总结一下在这个项目中我的收获。

<!--more-->

### 1. 生命周期

在Rust中，任何变量，对象都有它的生命周期与作用域，刚开始看Rust的书时，会想是不是只有**指针**需要标注，它才有生命周期，其实不是的，任何变量都有生命周期，在作用域闭合时，生命周期也就结束了，而内存也会被释放。这就是Rust通过生命周期来管理内存的机制。

变量的作用域可以从前一个转移到后一个，也就是`move`, 转移后前一个作用域就不再持有这个变量，这就是所谓的所有权，这个变量是属于这个作用域，在作用域内就可以使用，而在`move`出作用域后就无法使用了。比如一个函数的调用，对象在传入后，对象的作用域就转移到新的函数作用域中。那我们如果希望在原来的作用域中继续使用这个变量，就涉及到`Copy` trait与`Clone` trait了，如果对象实现了`Copy` trait，那么这个变量就可以在原来的作用域中继续使用。如果对象实现了`Clone` trait，那么这个变量可以在新的作用域中通过clone方法来获取。无论是`Copy` trait还是`Clone` trait，相当于把对象复制了一份，在新的作用域中可以使用。

有时候我们并不希望每次函数调用都把对象复制一份，并且在函数中也不需要持有对象的所有权，这个时候就可以通过`&`来**借用**对象，通过`&mut`来**可变借用**对象。初看Rust的书，我会觉得这不就是指针吗，哪个语言没有这个概念呀，但是真正理解了生命周期与所有权后，就会发现这确实就是**借用**，我不给你所有权，你必须在我的作用域内使用，这就是借用。

然后再来理解下一个对象不能同时存在**可变借用**与**不可变借用**，也就比较好理解了，如果可变借用被修改了，那同时存在的不可以变借用到底是变了还是没变呢，如果都存在就是矛盾的。

说完了借用，再来看看智能指针，我希望在多个作用域中使用对象，并且都有所有权，那就可以使用`Rc<T>`，它实现了引用计数，当对象被Rc所持有时，引用计数加1，当离开作用域后，引用计数减1，当引用计数为0时内存被释放。`Rc<T>`是不是线程安全的，所以我们往往使用`Arc<T>`，它实现了原子的引用计数，所以是线程安全的。这2个智能指针是通过 `Drop` trait来实现引用计数的变更。

`Rc<T>`与`Arc<T>`如果直接包裹对象，那对象就是不可变的，为了实现可变借用，需要使用`RefCell<T>`包裹对象，通过`borrow_mut()`来获取可变借用。然而`RefCell<T>`也不是线程安全的，所以需要使用`Mutex<T>`或者`RWLock<T>`来包裹对象，通过`lock()`获取可变借用。

说完了生命周期，所有权，借用，智能指针，举几个我在PingSIX中的实际代码，来帮助理解一下：

```rust
/// Proxy load balancer.
///
/// Manages the load balancing of requests to upstream servers.
pub struct ProxyUpstream {
    pub inner: config::Upstream,
    lb: SelectionLB,

    runtime: Option<Runtime>,
    watch: Option<watch::Sender<bool>>,
}

impl ProxyUpstream {
    pub fn new_with_health_check(upstream: config::Upstream, work_stealing: bool) -> Result<Self> {
        let mut proxy_upstream = Self::try_from(upstream)?;
        proxy_upstream.start_health_check(work_stealing);
        Ok(proxy_upstream)
    }

    /// Starts the health check service, runs only once.
    pub fn start_health_check(&mut self, work_stealing: bool) {
        if let Some(mut service) = self.take_background_service() {
            // Create a channel for watching the health check status
            let (watch_tx, watch_rx) = watch::channel(false);
            self.watch = Some(watch_tx);

            // Determine the number of threads for the service
            let threads = service.threads().unwrap_or(1);

            // Create a runtime based on the work_stealing flag
            let runtime = self.create_runtime(work_stealing, threads, service.name());

            // Spawn the service on the runtime
            runtime.get_handle().spawn(async move {
                service.start_service(None, watch_rx).await;
                info!("Service exited.");
            });

            // Set the runtime lifecycle with ProxyUpstream
            self.runtime = Some(runtime);
        }
    }

    fn create_runtime(&self, work_stealing: bool, threads: usize, service_name: &str) -> Runtime {
        if work_stealing {
            Runtime::new_steal(threads, service_name)
        } else {
            Runtime::new_no_steal(threads, service_name)
        }
    }

    fn stop_health_check(&mut self) {
        if let Some(tx) = self.watch.take() {
            let _ = tx.send(true);
        }
    }

    ...
}
```

`ProxyUpstream`是反向代理的上游对象，在new这个对象的同时会创建一个后台任务来执行健康检查，可以看到健康检查在这里是通过`tokio`的`Runtime`来执行的。在Golang中我可能会通过`go`关键字来创建一个goroutine来执行后台任务就不管了，然而在Rust这里我必须在`spawn`后台任务后让`ProxyUpstream`持有`Runtime`的生命周期，否则后台任务会因为作用域结束而提前退出。在`ProxyUpstream`持有`Runtime`后，只有当`ProxyUpstream`对象被drop后，后台任务才会结束。

```rust
impl Drop for ProxyUpstream {
    /// Stops the health check service if it exists.
    fn drop(&mut self) {
        self.stop_health_check();

        // 确保其他资源如 runtime 被释放
        if let Some(runtime) = self.runtime.take() {
            // 获取 runtime 的 handle
            let handler = runtime.get_handle().clone();

            // 使用 handler 执行关闭逻辑
            handler.spawn_blocking(move || {
                runtime.shutdown_timeout(Duration::from_secs(1));
            });

            info!("Runtime shutdown successfully.");
        }
    }
}
```

再来看看`Drop` trait的实现，在`ProxyUpstream`对象被释放的同时我们希望停止健康检查，这个时候还需要主动关闭`Runtime`，否则运行中的`Runtime`会因为作用域结束而panic。

再来看一个可变借用与不可变借用冲突的例子：

```rust
#[async_trait]
impl ServeHttp for AdminHttpApp {
    async fn response(&self, http_session: &mut ServerSession) -> Response<Vec<u8>> {
        http_session.set_keepalive(None);

        if validate_api_key(http_session, &self.config.api_key).is_err() {
            return Response::builder()
                .status(StatusCode::FORBIDDEN)
                .body(Vec::new())
                .unwrap();
        }

        let req_header = http_session.req_header();
        let path = req_header.uri.path().to_string();
        let method = req_header.method.clone();

        //   let (path, method) = {
        //       let req_header = http_session.req_header();
        //       (req_header.uri.path().to_string(), req_header.method.clone())
        //   };

        match self.router.at(&path) {
            Ok(Match { value, params }) => match value.get(&method) {
                Some(handler) => {
                    let params: BTreeMap<String, String> = params
                        .iter()
                        .map(|(k, v)| (k.to_string(), v.to_string()))
                        .collect();
                    match handler.handle(&self.etcd, http_session, params).await {
                        Ok(resp) => resp,
                        Err(e) => Response::builder()
                            .status(StatusCode::BAD_REQUEST)
                            .body(e.to_string().into_bytes())
                            .unwrap(),
                    }
                }
                None => Response::builder()
                    .status(StatusCode::METHOD_NOT_ALLOWED)
                    .body(Vec::new())
                    .unwrap(),
            },
            Err(_) => Response::builder()
                .status(StatusCode::NOT_FOUND)
                .body(b"Not Found".to_vec())
                .unwrap(),
        }
    }
}
```

`http_session`是一个可变借用，然后在获取`path`与`method`时，`http_session`会变成不可变借用来调用`req_header()`，然而在获取`path`与`method`后，`http_session`又需要被修改，所以这里同时存在了可变借用与不可变借用，Rust编译器会提示错误。那怎么修改呢？

```rust
#[async_trait]
impl ServeHttp for AdminHttpApp {
    async fn response(&self, http_session: &mut ServerSession) -> Response<Vec<u8>> {
        ...

        let (path, method) = { // 创建一个单独的作用域
            let req_header = http_session.req_header();
            (req_header.uri.path().to_string(), req_header.method.clone())
        };

        ...
    }
}
```

我们给`req_header`创建一个单独的作用域，这样`http_session`就变成不可变借用，在作用域结束时，`req_header`也跟着释放了。这用就不会同时存在可变借用与不可变借用冲突了。

### 2. 函数式编程

以往在写Python或Golang的过程中基本上没怎么用到函数式编程，Python中虽然有map、filter等函数，但是也只是在简单的场景下使用。而Rust中函数式编程是必不可少的，它让代码更简洁、更优雅。在写PingSIX的过程中，刚开始我也没怎么用到函数式编程，一直就式老老实实的if else，match等等。这样代码其实也没问题，但是会让代码看起来很长，所以我就把我的代码帖到ChatGPT问问它如何能让代码更加简洁，就有了下面的这些代码：

```rust
impl Upstream {
    fn validate_upstream_host(&self) -> Result<(), ValidationError> {
        if self.pass_host == UpstreamPassHost::REWRITE {
            self.upstream_host.as_ref().map_or_else(
                || Err(ValidationError::new("upstream_host_required_for_rewrite")),
                |_| Ok(()),
            )
        } else {
            Ok(())
        }
    }
}
```

这是一个简单是否为`Option`的判断，如果为`None`则返回错误，否则返回`Ok(())`。

```rust
#[async_trait]
impl ServiceDiscovery for DnsDiscovery {
    /// Discovers backends by resolving DNS names to IP addresses.
    async fn discover(&self) -> Result<(BTreeSet<Backend>, HashMap<u64, bool>)> {
        let name = self.name.as_str();
        log::debug!("Resolving DNS for domain: {}", name);

        let backends: BTreeSet<Backend> = self
            .resolver
            .lookup_ip(name)
            .await
            .or_err_with(InternalError, || {
                format!("DNS discovery failed for domain: {}", name)
            })?
            .iter()
            .map(|ip| {
                let addr = SocketAddr::new(ip, self.port as u16).to_string();

                // Creating backend
                let mut backend = Backend::new_with_weight(&addr, self.weight as usize).unwrap();

                // Determine if TLS is needed
                let tls = matches!(self.scheme, UpstreamScheme::HTTPS | UpstreamScheme::GRPCS);

                // Create HttpPeer
                let mut peer = HttpPeer::new(&addr, tls, self.name.clone());
                if matches!(self.scheme, UpstreamScheme::GRPC | UpstreamScheme::GRPCS) {
                    peer.options.alpn = ALPN::H2;
                }

                // Insert HttpPeer into the backend
                assert!(backend.ext.insert::<HttpPeer>(peer).is_none());

                backend
            })
            .collect();

        // Return backends and an empty HashMap for now
        Ok((backends, HashMap::new()))
    }
}
```

这是一串DNS解析与转换的代码，说实话我的脑子里还没形成这种函数式编程的定式思维，虽然我看的懂，但是真正写的时候我还是不会直接想到这么写，幸好有ChatGPT帮我解决了这个问题。我想写的越多也就越熟悉了吧。

### 3. 多态

在Rust里面写多态有2中选择，一种使用trait，一种是用enum，我们分别来看下。

#### enum

```rust
enum SelectionLB {
    RoundRobin(LB<RoundRobin>),
    Random(LB<Random>),
    Fnv(LB<FVNHash>),
    Ketama(LB<KetamaHashing>),
}

impl TryFrom<config::Upstream> for SelectionLB {
    type Error = Box<Error>;

    fn try_from(value: config::Upstream) -> Result<Self> {
        match value.r#type {
            config::SelectionType::RoundRobin => {
                Ok(SelectionLB::RoundRobin(LB::<RoundRobin>::try_from(value)?))
            }
            config::SelectionType::Random => {
                Ok(SelectionLB::Random(LB::<Random>::try_from(value)?))
            }
            config::SelectionType::Fnv => Ok(SelectionLB::Fnv(LB::<FVNHash>::try_from(value)?)),
            config::SelectionType::Ketama => {
                Ok(SelectionLB::Ketama(LB::<KetamaHashing>::try_from(value)?))
            }
        }
    }
}
```

这是一个包装了多种负载均衡算法的枚举，在`try_from()`方法中根据不同的类型创建对应的负载均衡算法。在使用时，我需要根据不同enum的variant进行match，然后调用对应的负载均衡算法。


```rust
impl ProxyUpstream {
    /// Selects a backend server for a given session.
    pub fn select_backend<'a>(&'a self, session: &'a mut Session) -> Option<Backend> {
        let key = request_selector_key(session, &self.inner.hash_on, self.inner.key.as_str());
        log::debug!("proxy lb key: {}", &key);

        let mut backend = match &self.lb {
            SelectionLB::RoundRobin(lb) => lb.upstreams.select(key.as_bytes(), 256),
            SelectionLB::Random(lb) => lb.upstreams.select(key.as_bytes(), 256),
            SelectionLB::Fnv(lb) => lb.upstreams.select(key.as_bytes(), 256),
            SelectionLB::Ketama(lb) => lb.upstreams.select(key.as_bytes(), 256),
        };

        if let Some(backend) = backend.as_mut() {
            if let Some(peer) = backend.ext.get_mut::<HttpPeer>() {
                self.set_timeout(peer);
            }
        }

        backend
    }
}
```

这样我们就可以隐藏不同负载均衡算法的细节，对外提供统一的调用入口来实现多态。

在Pingora中处理http1与http2的差异时也使用enum来实现多态。

#### trait

```rust
/// Hybrid service discovery.
///
/// Combines static and DNS-based service discovery.
#[derive(Default)]
pub struct HybridDiscovery {
    discoveries: Vec<Box<dyn ServiceDiscovery + Send + Sync>>,
}
```

为了包装DNS的服务发现与静态服务发现，我定义了`HybridDiscovery`同时处理这两种服务发现，它持有一个`Vec<Box<dyn ServiceDiscovery + Send + Sync>>`的成员变量。这里通过Box包裹的trait object来实现多态。当然为了支持async trait，还要加上`Send + Sync`。

### 4. 局部可变性

在我使用etcd来实现PingSix的资源实时加载时，遇到了这样的问题，对etcd client的使用必须是mut的，但是持有etcd client的对象又是不可变的，所以这里就必须包装出局部可变的etcd client，代码如下：

```rust
pub struct EtcdClientWrapper {
    config: Etcd,
    client: Arc<Mutex<Option<Client>>>,
}

impl EtcdClientWrapper {
    pub fn new(cfg: Etcd) -> Self {
        Self {
            config: cfg,
            client: Arc::new(Mutex::new(None)),
        }
    }

    async fn ensure_connected(
        &self,
    ) -> Result<Arc<Mutex<Option<Client>>>, Box<dyn Error + Send + Sync>> {
        let mut client_guard = self.client.lock().await;

        if client_guard.is_none() {
            log::info!("Creating new etcd client...");
            *client_guard = Some(create_client(&self.config).await?);
        }

        Ok(self.client.clone())
    }

    pub async fn get(&self, key: &str) -> Result<Option<Vec<u8>>, Box<dyn Error + Send + Sync>> {
        let client_arc = self.ensure_connected().await?;
        let mut client_guard = client_arc.lock().await;

        let client = client_guard
            .as_mut()
            .ok_or("Etcd client is not initialized")?;

        let resp = client.get(self.with_prefix(key), None).await?;
        Ok(resp.kvs().first().map(|kv| kv.value().to_vec()))
    }

    pub async fn put(&self, key: &str, value: Vec<u8>) -> Result<(), Box<dyn Error + Send + Sync>> {
        let client_arc = self.ensure_connected().await?;
        let mut client_guard = client_arc.lock().await;

        let client = client_guard
            .as_mut()
            .ok_or("Etcd client is not initialized")?;

        client.put(self.with_prefix(key), value, None).await?;
        Ok(())
    }

    pub async fn delete(&self, key: &str) -> Result<(), Box<dyn Error + Send + Sync>> {
        let client_arc = self.ensure_connected().await?;
        let mut client_guard = client_arc.lock().await;

        let client = client_guard
            .as_mut()
            .ok_or("Etcd client is not initialized")?;

        client.delete(self.with_prefix(key), None).await?;
        Ok(())
    }

    fn with_prefix(&self, key: &str) -> String {
        format!("{}/{}", self.config.prefix, key)
    }
}
```

就如我在第一节的生命周期中所说的，为了局部可变，我们必须使用Arc<Mutex>包裹，这样在任何地方都可以通过lock()来获取可变引用。

### 总结

在写PingSIX之前，我写过一些Rust的小项目，比如一些http接口的调用，比如一些TCP协议的解析。但是都没有系统性的来使用Rust，所以一直有计划用Rust来写一个API网关，然后这一个多月下来，我对Rust可以说有了一定的理解吧，虽然像`Send + Sync`这种东西还是不太理解，但是至少知道怎么用了。在比如像`FnOnce`，还有`Pin`这些也都还不太了解，但是可以说现在用Rust写点项目还是可以的。

PingSIX虽然只实现了APISIX的一个子集，有很多功能的缺失，比如被动的健康检查，动态的TLS支持，还有很多插件等，但是大的框架其实已经搭好了，剩下的其实都是体力活了。所以暂时先告一段落吧。

通过这个项目我对Pingora的各个crate都有了深度使用，当然还有一些功能也是没有覆盖到，比如下面的这些：

dynamic ssl：<https://github.com/cloudflare/pingora/blob/main/pingora/examples/server.rs#L77>

cache：<https://github.com/cloudflare/pingora/blob/main/pingora-proxy/tests/utils/server_utils.rs#L393>

这些都还没有文档，所以暂时先不提了。

在开发这个项目的过程中，我频繁使用ChatGPT，深刻的感受到ChatGPT对于程序员的效率的提升，项目中的所有代码都经过了ChatGPT的Review，AI不会淘汰程序员，只会淘汰不会使用AI的程序员。