---
layout: post
title: 基于 gRPC 的服务发现及相关核心组件、交互分析
tags: 源码相关
excerpt: resovler.Resolver ...
stickie: true
---

> 基于v1.64.0；

# 服务发现

&emsp;&emsp;可自定义解析器；

### Resolver 解析器
负责：
- 服务发现的实现；
- 和命名服务（注册中心）通信，**实时**获取服务器列表（或者说会处理变更信息）；
- 将更新结果，及时发送给 Balancer，用于更新 gRPC 长连接池（Connection Pool）；

<p><img src="https://acceleratorssr.github.io/image/resolver.png" alt="请求流程"></p>

&emsp;&emsp;一般注册中心也分为两种通知模式，PULL 或者 PUSH 模式，如 DNS 为 PULL 模式，而 etcd 属于PUSH模式；

`resolver.Build()` 在 grpc 客户端发起连接时被调用，即：

`NewClient(target string, opts ...DialOption) (conn *ClientConn, err error)`

&emsp;&emsp;gRPC 会选择一个解析器解析目标字符串 target，target 会被转换为 Target 类型，字段 url就包含了 Scheme、Host、Query等关键信息；

#### 自定义实现的思路

> 详细请见 [implement](https://acceleratorssr.github.io/2024/10/20/grpc-self-implement.html)

关键在于实现 `Builder` 和 `Resolver` 两个接口；

&emsp;&emsp;`Builder` 接口用于创建 Resolver，可提供自定义的服务发现实现，然后将其注册到 gRPC 中，通过 url 中的 Scheme 来标识；

&emsp;&emsp;`Resolver` 接口则负责提供服务发现功能，当服务列表发生变化时，通过 ClientConn 回调接口 `UpdateState` 通知上层，更新客户端连接的可用服务地址列表，进而更新负载均衡器和实际客户端连接；

### resolver 核心接口

```go
// ClientConn 包含 resolver 的回调
// 用于通知 gRPC ClientConn 发生了任何更新
type ClientConn interface {
	// UpdateState 会对应更新 ClientConn 的状态，被 resolver 调用
	UpdateState(State) error
    
	// 解析服务地址时遇到的错误，回调给 ClientConn
	ReportError(error)

	// 弃用
	NewAddress(addresses []Address)
	// 解析json格式的grpc服务配置
	ParseServiceConfig(serviceConfigJSON string) *serviceconfig.ParseResult
}

// 表示客户端要连接的服务器信息，由 resolver 提供
type Address struct {
	Addr string
	ServerName string
	Attributes *attributes.Attributes
	BalancerAttributes *attributes.Attributes

		// 弃用：改用Attributes
	Metadata any
}

// 用于创建一个 resolver
type Builder interface {
	// ClientConn 会接收 Resolver 返回的地址更新
	// 调用时机：
	// 初次建立连接
	// 客户端重新连接：连接中断时，gRPC 客户端会调用 ResolveNow 尝试重新解析
	// resolver.Resolver.ResolveNow() 手动触发解析过程
	// 此处可开启单独的 goroutine，进行 list-watcher 逻辑
	Build(target Target, cc ClientConn, opts BuildOptions) (Resolver, error)
	// 解析出使用的协议类型，如 etcd等
	Scheme() string
}

// watcher
type Resolver interface {
	// 连接出现异常时，调用该方法重新实现一次服务发现
	ResolveNow(ResolveNowOptions)
	Close()
}

type Target struct {
	URL url.URL // 内含 scheme 等信息
}
```

### 从 grpc.NewClient 可看 gRPC 做了什么初始化工作

<p><img src="https://acceleratorssr.github.io/image/grpcClientInit.png" alt="初始化流程"></p>

```go
func NewClient(target string, opts ...DialOption) (conn *ClientConn, err error) {
	... // 初始化 ClientConn

	... // 全局的选型初始化

		// 应用传入的 opt
	for _, opt := range opts {
		opt.apply(&cc.dopts)
	}

	// 链式应用客户端拦截器，此处分为一元和流式两种
	chainUnaryClientInterceptors(cc)
	chainStreamClientInterceptors(cc)

	// 验证TLS
	if err := cc.validateTransportCredentials(); err != nil {
		return nil, err
	}

	// 解析服务配置（负载均衡的配置就在这被解析出来，即获取 balancer.Builder）
	if cc.dopts.defaultServiceConfigRawJSON != nil {
		scpr := parseServiceConfig(*cc.dopts.defaultServiceConfigRawJSON)
		if scpr.Err != nil {
			return nil, fmt.Errorf("%s: %v", invalidDefaultServiceConfigErrPrefix, scpr.Err)
		}
		cc.dopts.defaultServiceConfig, _ = scpr.Config.(*ServiceConfig)
	}
	cc.mkp = cc.dopts.copts.KeepaliveParams

	// 将 ClientConn 注册到 channelz 里
	// 用于监控连接和通道状态
	cc.channelzRegistration(target)

	// 解析目标及寻找解析器
	if err := cc.parseTargetAndFindResolver(); err != nil {
		channelz.RemoveEntry(cc.channelz.ID)
		return nil, err
	}

/* ------------------------------------------------------------------------
注：在自定义 resolver 的情况下，进入 parseTargetAndFindResolver 后，只需要先解析出
target 的 scheme，又因为当初注册 resovler 的 builder 时，使用的 key 是自定义的 scheme，
故在获取 target 的 scheme 后即可直接获取自定义的 resolver，直接就返回到 NewClient；

parseTargetAndFindResolver 的后部分是在没有通过 scheme 找到对应 resolver，或者
没有 scheme 的情况下，使用默认的直连模式；
------------------------------------------------------------------------ */

	// TLS相关
	if err = cc.determineAuthority(); err != nil {
		channelz.RemoveEntry(cc.channelz.ID)
		return nil, err
	}

	// 创建 连接状态管理器
	cc.csMgr = newConnectivityStateManager(cc.ctx, cc.channelz)
	// 创建出负载均衡器中的选择器
	cc.pickerWrapper = newPickerWrapper(cc.dopts.copts.StatsHandlers)

	// 空闲连接状态
	// 创建 resolver 和 balancer 的包装器（wrapper）
	cc.initIdleStateLocked()
	cc.idlenessMgr = idle.NewManager((*idler)(cc), cc.dopts.idleTimeout)
	return cc, nil
}
```

### 建立连接并完成对应元数据、状态传递

（完成 `NewClient` 后）发请求前，建立 etcd 连接：

&emsp;&emsp;`start()` 后，通过 `serializer.Schedule` 安全调用 `build()`（获取 `resolver.Resolver`）

&emsp;&emsp;在 **build** 内调用 **resolver wrapper** 的 `UpdateState` 方法，传递 **State** 到 `ClientConn` 结构体上，

&emsp;&emsp;以 `ClientConn` 为传递者，再将 **State** 传递给 `balancer wrapper`，通过 `serializer.Schedule` 安全传递到 `balancer.Balancer` 的实现里（如原本为 **baseBalaner**），并更新对应信息，如调用 `NewSubConn` 构造出 **addrConn**（即真实的连接，也被 **ClientConn** 追踪管理，其负责管理多个服务节点的地址等元数据，返回时会被包装为实现 **SubConn** 接口的 **acBalancerWrapper**），拿到被 **wrapper** 后的 **addrConn**，即可调用 **SubConn** 的 `Connect` 方法建立连接（开启goroutine，不会阻塞）；

<br>

发起第一次 rpc 请求（或**节点变化**）后：

&emsp;&emsp;直接调用自定义的 `resolver.Builder` 的 **build** 方法，解析节点，并将更新后的节点信息传递给 **resolver wrapper** 的 `UpdateState` 方法，后面的流程和上述步骤相同了，最后都是到 **baseBalaner** 内更新连接等信息；

&emsp;&emsp;而 HTTP2.0 长连接也都是通过 **ClientConn**（通过 **balancer wrapper** 传递） 的 `newAddrConnLocked` 方法建立的（初始连接都为 idle状态），并将连接信息传递回 baseBalaner，拿到连接信息后，先通过拿到的 balancer.SubConn 建立连接（Connect），再去更新负载均衡器的选择器（build）；

&emsp;&emsp;如果光看调用流程，初见可能有点晕，但仔细分析也不算复杂，个人认为可以先从几个核心结构体的关系开始入手，本质上其实就分为**连接模块**、**负载均衡模块**、**解析模块**，gRPC 为了简化模块间的传输操作，将 **balancer** 和 **resolver** 均使用**装饰器模式**（wrapper）包装了一层，并都额外实现了 **clientconn** 接口，通过这个接口，负载均衡器和解析器的通讯就主要是依赖 `UpdateState(State)` 方法传递元数据的（注：State 包含的数据见下方引用部分），个人认为可以从韦恩图一眼看出他们的关系（跳过 builder 的关系）：

<p><img src="https://acceleratorssr.github.io/image/call.png" alt="核心结构的关系"></p>

故上方源码调用链就容易理解点了，核心目标就是**两个**方向的 数据传递：
- **事件**： resolver 解析到服务节点发生变化；**目标**：通知 balancer 重新维护连接状态；
- **事件**：需要获取 RPC 请求的目标地址；**目标**：picker 的最新信息会同步到 ClientConn，并由 ClientConn 获取连接信息，通过 transport 发起真实的 rpc 请求；

> &emsp;&emsp;Balancer 传递的 State 包含两个字段，分别是：标识负载均衡器的连接状态，选择器的实现（由ClientConn 用于选择连接进行 rpc 调用）；
>
> &emsp;&emsp;Resolver 传递的 State 包含四个字段，目前分别是：addresses 解析出的地址切片，endpoints 服务节点的信息切片，服务器的配置信息，和 attributes 额外信息；
> &emsp;&emsp;官方表明要用 endpoints 替换掉此处的 addresses ，虽然观察 resolver wrapper 处的源码可以发现当其传递信息到 ClientConn 时，endpoints 为空时会获取一份 addresses 副本，但起码目前 `1.64.0` 的版本下，官方给的默认 balancer.Balancer 实现 baseBalancer 中，依然使用 addresses 建立连接等操作，并没有使用 endpoints；（也可能在其他地方使用，但至少通过 ClientConn 传递建立连接的数据时就只用了 addresses）

### 传递服务节点元数据及构建核心结构

（初始化解析器）`获取可用服务节点` 并 `通知上游及负载均衡器` 的调用链路流程图（省去处理数据）（**核心中的核心**）：

<p><img src="https://acceleratorssr.github.io/image/key.png" alt="核心变更节点的流程"></p>

> CallbackSerializer 结构体：提供同步方式调度回调函数，保证顺序即线性化，以FIFO执行任务；
> 
> 如 在此处使用的 Schedule 方法，就是将传入的方法放入缓冲区，并允许调度；
> 
> 内部开启的 goroutine 会一直处理回调函数；

### 发起 rpc 调用的流程

&emsp;&emsp;发起调用的流程（体现上文的关系，即 ClientConn 会从 picker 取出 SubConn 并建立连接）：

&emsp;&emsp;由 **ClientConn** 调用 `Invoke`（起点），再调用 grpc 包下的 `invoke` 方法，初始化相关参数，调用 `newClientStream` 方法实例化一个新的客户端流，设置相关请求信息，再到 **clientSteam** 的 `newAttemptLocked` 方法尝试发起调用；

&emsp;&emsp;再进入 **csAttempt** 的 `getTransport` 方法，调用 **ClientConn** 的 `getTransport` 方法，拿到 **picker wrapper**，从而可以调用自定义实现的 **pick**，拿到 **SubConn** 后，返回 **csAttempt** 处调用 `newStream` 方法创建实际的 RPC 流，最后通过 **clientSteam** 的 `sendMsg` 和 `RecvMsg` 即可发送请求并接收响应；

### 相关包

base 包，即原本 gRPC 的默认 balancer.Balancer 实现，可被替换：

```go
// 首先，需要通过 init 将 balancer.Builder 注册给 base，
// 核心把 pickerBuilder 传给base
// 所以其实在这里也能分析出：
// 可选择两种 implement：
//     1）直接实现整个 balancer.Builder 和 balancer.Balancer（包括picker），即替换掉 base 包；
//      
//     2）只实现 PickerBuilder 和 picker，仅替换负载均衡的选择器，但连接等由 base 包管理
func NewBalancerBuilder(name string, pb  PickerBuilder, config Config) balancer.Builder {
	return &baseBuilder{
		name:          name,
		pickerBuilder: pb,
		config:        config,
	}
}

// 被 picker 的Build方法依赖
type PickerBuildInfo struct {
	// ReadySCs is a map from all ready SubConns to the Addresses used to
	// create them.
	ReadySCs map[balancer.SubConn]SubConnInfo
}

待续...
```

attributes 包，用于存储**不可变**的额外信息；
```go
// 是一个 不可变 结构，用于存储和检索 泛型的 KV对，
// key 必须是可哈希的结构
// 注：value 需要实现 Equal(o any) bool，否则会使用 == 进行比较
// 不可变：简化并发处理并避免数据竞争
type Attributes struct {
	m map[any]any
}

// 每次都会返回新的 Attributes 实例
// 通过修改本地的 map 避免数据竞争
// 类似于函数式编程
func (a *Attributes) WithValue(key, value any) *Attributes {
	if a == nil {
		return New(key, value)
	}
	n := &Attributes{m: make(map[any]any, len(a.m)+1)}
	for k, v := range a.m {
		n.m[k] = v
	}
	n.m[key] = value
	return n
}
```

平滑切换，待续...

channelz，待续...