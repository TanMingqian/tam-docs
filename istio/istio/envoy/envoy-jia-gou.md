# envoy架构

### istio和envoy的工作流程

**用户端**，通过创建服务治理的规则（VirtualService、DestinationRule等资源类型），存储到ETCD中

**istio控制平面**中的**Pilot**服务监听上述规则，转换成envoy可读的规则配置，通过**xDS**接口同步给各envoy

**envoy**通过xDS获取最新的配置后，动态reload，进而改变流量转发的策略&#x20;

### 认识envoy <a href="#2-e3-80-81-e8-ae-a4-e8-af-86envoy" id="2-e3-80-81-e8-ae-a4-e8-af-86envoy"></a>

Envoy 是为云原生应用设计的代理。

网络代理程序的流程：比如作为一个代理，首先要能获取请求流量，通常是采用监听端口的方式实现；其次拿到请求数据后需要对其做微处理，例如附加 Header 或校验某个 Header 字段的内容等，这里针对来源数据的层次不同，可以分为 L3/L4/L7，然后将请求转发出去；转发这里又可以衍生出如果后端是一个集群，需要从中挑选一台机器，如何挑选又涉及到负载均衡等。&#x20;

### 资源对象

**listener** : Envoy 的监听地址。Envoy 会暴露一个或多个 Listener 来监听客户端的请求。

**filter** : 过滤器。在 Envoy 中指的是一些“可插拔”和可组合的逻辑处理层，是 Envoy 核心逻辑处理单元。

**route\_config** : 路由规则配置。即将请求路由到后端的哪个集群。

**cluster** : 服务提供方集群。Envoy 通过服务发现定位集群成员并获取服务，具体路由到哪个集群成员由负载均衡策略决定。

### XDS

Envoy的启动配置文件分为两种方式：**静态配置和动态配置。**

静态配置是将所有信息都放在配置文件中，启动的时候直接加载。

动态配置需要提供一个Envoy的服务端，用于动态生成Envoy需要的服务发现接口，这里叫XDS，通过发现服务来动态的调整配置信息，Istio就是实现了v2的API。

**Envoy 接收到请求后，会先走 FilterChain，通过各种 L3/L4/L7 Filter 对请求进行微处理，然后再路由到指定的集群，并通过负载均衡获取一个目标地址，最后再转发出去。**

其中每一个环节可以静态配置，也可以动态服务发现，也就是所谓的 xDS。这里的 x 是一个代词，类似云计算里的 XaaS 可以指代 IaaS、PaaS、SaaS 等。

### 概念

#### Downstream

下游（downstream）主机连接到 Envoy，发送请求并或获得响应。

#### Upstream

上游（upstream）主机获取来自 Envoy 的链接请求和响应。

#### 监听器

除了过滤器链之外，还有一种过滤器叫监听器过滤器（Listener filters），它会在过滤器链之前执行，用于操纵连接的元数据。这样做的目的是，无需更改 Envoy 的核心代码就可以方便地集成更多功能。

每个监听器都可以配置多个过滤器链（Filter Chains），监听器会根据 filter\_chain\_match 中的匹配条件将流量转交到对应的过滤器链，其中每一个过滤器链都由一个或多个网络过滤器（Network filters）组成。这些过滤器用于执行不同的代理任务，如速率限制，TLS 客户端认证，HTTP 连接管理，MongoDB 嗅探，原始 TCP 代理等。

### 架构

<figure><img src="../../../.gitbook/assets/image (5).png" alt=""><figcaption></figcaption></figure>

###

