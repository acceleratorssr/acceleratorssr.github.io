---
layout: post
title: grpc 插件式自定义 Resolver & Balancer
tags: 项目核心思考
excerpt: 关于在项目中的实际应用过程及相关问题，侧重于 LB 方面；
stickie: true
---

> 基于 gRPC v1.64.0

# 原理部分

### balancer.Balancer 接口 自定义实现
&emsp;&emsp;核心在于自己实现 **balancer.Builder** 接口，在 balancer 的注册阶段将自定义的 **balancer.Builder** 通过 init 将其注册进去；

```go
func init() {
	balancer.Register(newBuilder())
}

// func newBuilder() balancer.Builder
```

&emsp;&emsp;此时自定义的 **balancer.Builder** 将会将三个参数组装进自实现 **balancer.Balancer** 的结构体：负载均衡的**名称**（用于使用时，用json进行匹配），自定义的 **PickerBuilder** ，和是否进行和服务节点心跳检测；

&emsp;&emsp;需要注意，**balancer.Balancer** 需要维护 **balancer.ClientConn** 等额外元数据，如 用于让 **ClientConn** 构造出 **addrConn**，并且将其封装为 **SubConn** 接口类型后，调用 `Connect` 方法建立连接；

&emsp;&emsp;核心文件：`balancer\balancer.go`、`resolver_wrapper.go`、`clientconn.go`、`balancer/base/balancer.go`（主要是看流程，这部分是要被自定义替换的）、`gracefulswitch.go`；

&emsp;&emsp;间接核心文件：`service_config.go`、`Config.go`；

#### 获取 balancer.Builder 的时机和位置：

&emsp;&emsp;在初始化客户端连接时，即 **grpc.NewClient**；

&emsp;&emsp;在 `gracefulswitch` 包下：

&emsp;&emsp; `func parseServiceConfig(js string) *serviceconfig.ParseResult` 解析服务配置，并将 json 转换为 结构体 `serviceconfig.ParseResult`，故在此获取了获取负载均衡的配置，

&emsp;&emsp;然后通过 `func ParseConfig(cfg json.RawMessage) (serviceconfig.LoadBalancingConfig, error)` 处，处理解析后的 json 配置，并通过负载均衡器的选择器所注册的名字，获取 **balancer.Builder**；

&emsp;&emsp;并将 **Builder** 存入 **lbConfig** 内的 **childBuilder** 字段，

&emsp;&emsp;最后返回到 **clientconn** ，将 **lbConfig** 存放到当前的 **clientconn** 的 **dopts** 字段中的 **defaultServiceConfig** 字段，以供后续使用；

#### 选择器的自定义实现
&emsp;&emsp;因为选择器只需要关注负载均衡策略的实现即可，实现 `PickerBuilder` 和 `Picker` 接口即可，`build` 方法会更新节点信息，并通知到 Picker，所以这里不用维护过多非业务逻辑相关的元数据；

&emsp;&emsp;picker 和 resolver 的初始化都是在 NewClient 方法内完成的，较为直接，因为他两直接维护在 ClientConn 结构体中，以接口的形式，故不展开了；

### resolver.Resolver

&emsp;&emsp;因为同样不需要维护什么特别的额外元数据（`resolver.ClientConn` 除外，因为无论是实现 resolver 还是 balancer，都需要这个接口和 **ClientConn** 进行通信，才能做到信息的传递）

&emsp;&emsp;首先实现 `resolver.Builder` 和 `resolver.Resolver` 两个接口；

`resolver.Builder`：
&emsp;&emsp;使用 etcd 作为解析器，故将 scheme 设置为 etcd；

&emsp;&emsp;在创建一个 Resolver 时，开启一个 goroutine 用于 List-watcher 机制，节点发生变化时，主动调用 `ResolveNow` 方法重新解析服务节点（此处就是从 etcd 中拿到服务节点消息），并且通过 **resovler.ClientConn** 的 `UpdateState` 方法通知 **ClientConn** 进行节点更新，通过 `updateResolverStateAndUnlock` 方法将 **节点列表等信息** 和 **balancer.Builder** 传入 **ccBalancerWrapper** 的 **updateClientConnState** 方法中；

&emsp;&emsp;在 `balancer_wrapper.go`（ccBalancerWrapper） 中，通过 `newCCBalancerWrapper` 方法，将上文获取到的 **balancer.Builder** 传入 `gracefulswitch` 包下的 `func NewBalancer(cc balancer.ClientConn, opts balancer.BuildOptions) *Balancer`，并提取出自定义的 **balancer.Balancer**，放入 **ccBalancerWrapper** 结构体中的 `balancer *gracefulswitch.Balancer`；

&emsp;&emsp;当 `updateClientConnState` 被 gRPC 调用**推送更新节点信息**到负载均衡器后，**调用自定义的 balancer 的 UpdateClientConnState 方法**；

&emsp;&emsp;后面自定义的 balancer 部分尚未完成，暂时搁置放入 todo 了，核心就是维护连接及连接池，并将两方（picker、ClientConn）的数据进行交互和通知；

# 项目实践 & 前提条件准备
&emsp;&emsp;背景：（gRPC 自定义实现 **resolver** 和 **balancer**）通过 **resolver** 的 **Builder** 的 `build` 方法返回 **resolver** 时，使用 etcd 的 `watch` 机制监控服务节点的 **metadata** 的变化（即 及时获取负载情况），当 **etcdResolver** （实现 **resovler** 的结构体）接收到变更事件后，通过 **attributes** 传递 **metadata**，每次必须调用 **UpdateState** 才能将 **attributes** 传递更新到 **PickerBuilder** 的 `build` 方法中，以此更新 **picker** 的数据从而改变负载均衡策略，

&emsp;&emsp;**目标**：使服务节点将自己的负载情况，更新/通知给 **picker** 进行负载均衡策略的判断；

思考后得出 4 种方案：
1. 如背景所提及的，使用代理（etcd）获取服务节点信息，最后传递到 **balancer.Balancer** 上改变连接状态，但是**最重要的问题**是，grpc 在传递 **attributes** 的过程中，gRPC 会将 **metadata** 的改变视为连接的改变（即使 addr 没有变化），会断开当前连接并重新建立（具体信息判断如下）导致性能的不必要开销；
   
&emsp;&emsp;根据源码可看出，**baseBalancer**（**balancer.Balancer** 的实现者，可被替换）是通过 a 这个整体（**Address**）判断连接是否变化的，导致只是 metadata 变化，但 addr 没有变化的前提下，依然会将其判断为连接更新，即**重复建立连接**；

```go
// 核心部分：
// balancer\base\balancer.go 	baseBalancer
// clientconn.go 				addrConn
// transport.go 

type Address struct {
    Addr               string
    ServerName         string
    Attributes         *attributes.Attributes
    BalancerAttributes *attributes.Attributes
    Metadata           any
}

// 建立
for _, a := range s.ResolverState.Addresses {
    addrsSet.Set(a, nil) // 标记已有连接
    if _, ok := b.subConns.Get(a); !ok {
        ...
        sc, err := b.cc.NewSubConn([]resolver.Address{a}, opts) // 构建连接信息，包装 addrConn
        ...
        sc.Connect() // 建立真实HTTP2.0连接
	}
}

// 移除旧连接
for _, a := range b.subConns.Keys() {
    sci, _ := b.subConns.Get(a)
    sc := sci.(balancer.SubConn)
    if _, ok := addrsSet.Get(a); !ok {
        sc.Shutdown()
        b.subConns.Delete(a)
	}
}
```

&emsp;&emsp;利用 `netstat -a -n -o | findstr :9205` 可发现连接被重复建立了，具体来说，客户端主动断开和服务节点的联系后，重新建立一次新连接：

> - -a：显示所有连接和侦听端口
> - -n：以数字形式显示地址和端口号（不解析为主机名）
> - -o：显示每个连接的PID

> 注真实ip替换为了 10.0.0.1；
```powershell
1） 
TCP 10.0.0.1:12345 10.0.0.1:54321 ESTABLISHED 5616
TCP 10.0.0.1:54321 10.0.0.1:12345 ESTABLISHED 31500

2） 
TCP 10.0.0.1:54321 10.0.0.1:12345 TIME_WAIT 0

3） 
TCP 10.0.0.1:12345 10.0.0.1:56789 ESTABLISHED 5616
TCP 10.0.0.1:54321 10.0.0.1:12345 TIME_WAIT 0
TCP 10.0.0.1:56789 10.0.0.1:12345 ESTABLISHED 31500
```
```go
// 重新建立连接：
conn, err := dial(connectCtx, opts.Dialer, addr, opts.UseProxy, opts.UserAgent)
```
&emsp;&emsp;由此可见这个方案有比较大的性能不必要开销，不建议使用；

2. 第二种方法就是在第一种的基础上，将 gRPC 原本 **balancer.Balancer** 的实现者 **baseBalancer** 替换为自定义的实现，换句话说核心是替换掉 `UpdateClientConnState` 方法（或者利用装饰器重写），在收到服务节点变化的时候，会以 **addr** 作为更新节点连接的判断依据，并且保留通知 **picker** （即重新build）等其他重要逻辑；
3. 基于 **ORCA** （Open Request Cost Aggregation）开放标准，实现服务端的指标反馈架构；
4. 
gRPC 提供了两种指标上报机制（当然也可以自定义）：

1、按照 **请求** 的粒度上报：当 rpc 完成后，服务端后将自定义的指标附加到尾部元数据中（适合一元请求）；

2、**带外指标**上报（Out-of-band metrics reporting）：服务端会周期性向客户端推送服务端的指标数据（如CPU，内存利用率、qps等）
详见：
[orca package - google.golang.org/grpc/orca - Go Packages](https://pkg.go.dev/google.golang.org/grpc/orca)

[proposal/A51-custom-backend-metrics.md at master · grpc/proposal](https://github.com/grpc/proposal/blob/master/A51-custom-backend-metrics.md)

两种机制的使用案例：[grpc-go/examples/features/orca at master · grpc/grpc-go](https://github.com/grpc/grpc-go/tree/master/examples/features/orca)

4. 比较无脑，单纯在所有 rpc 响应的字段中，添加服务器的负载信息，但这种做法很麻烦，每次新的 rpc 都需要加上这个字段，不推荐；

# 负载均衡算法的选型调研 思考

&emsp;&emsp;在解决完实现原理层面的问题，就剩下选择器，即负载均衡的具体策略了，这方面 大致 的讨论可见[LB](https://acceleratorssr.github.io/2024/08/19/LB.html)；

## 轮询等静态无状态LB算法

&emsp;&emsp;总体来说没有特殊要求的情况下，可以优先考虑最简单但效果也不错的轮询算法，当服务节点是无状态的时候，随便水平扩展，轮询算法也不会受影响；

&emsp;&emsp;但在实际业务中，或者说在分布式数据系统无共享架构的环境下，如服务节点是**有状态**的，或者**复制滞后问题**较为严重时，如果继续使用轮询等无状态LB算法，那么性能大概率会惨不忍睹；

> 注：这里限制服务节点的有状态为：仅允许本地缓存，不引入二级缓存，如 redis；
> 且设定服务**处理过程**的特征是：高内存占用、CPU密集型（设定单纯为了突出问题的严重，其他像 session 等的处理其实也类似，都可以引入 共享二级缓存 解决）；

&emsp;&emsp;首先讨论节点**有状态**时，如在线视频中，当第一个用户请求新视频时，服务端会将该视频的片段索引化，即通过计算视频的关键帧等相关数据，以便在后续用户请求该视频时快速返回用于预期的视频片段，并将索引放入本地缓存以供其他用户使用，在这种情况下，如果用户连续的请求被分发到不同服务器上，那么每个服务器都需要独立建立相同视频的索引及重复缓存视频片段，造成冗余计算和资源浪费，结果上看缓存命中率下降，请求响应速度下降；

**核心的特征**是：
- 节点间各自**独立**维护缓存等状态信息；

&emsp;&emsp;再讨论**复制滞后问题**，首先明确这个概念的定义：

&emsp;&emsp;这个问题在分布式集群中可以说非常常见，是一个现象而不是一个不可接受的错误，最典型的场景就是在主从架构下，复制策略采用异步时，数据从主节点复制到各个从节点，并应用于从节点状态机的耗时不尽相同，甚至存在丢失的可能性；

> 注：个人认为此处讨论的主从架构类比到服务节点集群上，就是“共享”本地缓存的服务节点，但是从主从架构去分析比较通俗易懂，所以此处将讨论对象改为主从架构，但核心问题是相似的，即服务节点是有状态的，即使让节点间共享缓存也依旧存在问题；

&emsp;&emsp;但是在用户视角下主从集群就是**一个服务端**，在将数据写入后，期望服务端**整体的状态机**是持有最新数据的状态，但此时实际上只有主节点拥有最新的数据，而从节点则仅能保证最终一致性；

&emsp;&emsp;再回到主线上，当使用无状态的LB算法时，遇到的问题：本质上就是在达成最终一致性的窗口期内，用户发现了 **数据不一致** 的状态；

比较经典的场景如：
- **读自己的写**：用户写入数据后立刻读取，读取到的节点尚未复制到数据，但是在用户视角下：数据丢失了；
- **单调读**：用户多次读取数据，但发现数据“若隐若现”，同样因为读取到的部分节点尚未复制到数据；

> 注：此处还有一个复制滞后相关的问题：前缀一致性读，但他不是 LB算法 能解决的问题，故放在引用部分进行额外的解释：
> 
> 问题：用户 A 和 B 的对话信息，按照 A -> B 的顺序写入主节点，但是同步到从节点上时，部分节点的操作应用顺序混乱，变成 B -> A，与原意不符；
> 也可被理解为：操作的应用顺序混乱导致的问题，需要将原本的**最终**一致性（弱一致性）提升为**线性**一致性（强一致性），打个广告，我自己实现的 [raft](https://acceleratorssr.github.io/2024/10/21/raft.html) 算法即为读写线性一致性，可解决这个问题！

## 哈希类有状态LB算法

&emsp;&emsp;上文提及的问题，其实可以通过一个操作就可以解决（治标），及哈希类的 LB 算法；

&emsp;&emsp;从最基础的哈希取模（modulo hashing）介绍起，通过对请求的特征字段（或者说用于标识同类请求的部分）哈希处理后，对其取模，即可保证相同或者相似的操作会路由到相应的服务器上，但当服务节点变化频繁时，请求的**迁移率**高的吓人，简单计算可得出：假设请求的哈希结果较为均匀时，当有 `n` 个节点时，每新加 `k` 个节点，则有 `(1 - (1 / (n + k))` 个请求会被迁移到其他节点上，节点数量越多，迁移的比例越高；

&emsp;&emsp;进一步，引入一致性哈希算法即可解决迁移率高的问题，通过将服务节点均衡哈希到一个哈希环上（刻度一般对应 `0 ~ 2^32-1` 中的数值），在上文哈希请求的基础上，将哈希结果放于环上，并且顺时针（逆也行，不过一般约定俗成是顺吧）找到最近的一个服务节点，并将请求发送至这个服务节点上；比较清晰可看出它同样可以保证请求到确定的服务节点上；再简单计算下请求的**迁移率**：假设有 `n` 个节点，此时期望节点间距为 `1/n`，当新增 `k` 个节点时，期望间距变为 `1 / (n + k)`，此时通过图更直观，新节点加入环后，一定且也只会分走原本 一个 节点的数据，而分走的请求就是加入新节点后的期望间距：`1 / (n + k)`，相比哈希取模的方式，极大减少了请求的迁移；

> 一致性哈希算法还有一些优化，如 Ketama 算法，引入了虚拟节点，不展开讲了；

## 新问题接踵而至

&emsp;&emsp;其实从负载均衡的角度出发，在现实的业务场景中，哈希类算法的表现反而经常差于轮询等其他基础静态算法，乍一看有违常理，不是通过锁定请求路由的服务端，从而有效提升缓存命中率了吗？（性能方面考虑，略过 session 等）

&emsp;&emsp;但仔细一想，想要完全发挥哈希类算法的优势，有一个**重要的前提**是：哈希后的请求足够均匀；但现实场景中，有一句俗话或者说经验之谈：`百分之二十的数据占用了百分之八十的请求`，即**请求本身不均衡**，那么在单纯哈希的情况下，一个服务器的过载问题会极其严重；

&emsp;&emsp;并且考虑在当下云原生时代，往往集群都有扩容的调度策略，当服务器过载后，容易导致不必要的扩容（集群内有负载低的节点）；

&emsp;&emsp;并且不止这个问题，在现实中往往机器集群的硬件也很难做到统一配置，换句话说服务节点是**异质化**的，节点间的性能差异较大，这对于静态LB算法的限制是巨大的，以上讨论的LB算法（当然简单加权可以有所缓解，但不是重点，不展开讨论）都会受到不小的影响，

## 动态LB算法

&emsp;&emsp;既然上文提到，请求本身不均衡 和 服务节点异质化 两个主要的问题，抽象下他们的共性，那就是客户端不会感知服务端当下的处理能力，那么就容易想到 能不能让服务端以某种方式通知客户端自身的负载及处理能力呢？（具体的获取方式可回到上文提及的 4种方法）

&emsp;&emsp;答案显而易见，最直接的判断指标方法如：服务端的连接数、处理请求的时延，进一步涉及一些统计学等数学计算的算法，如 P2C 等；

> 注：如果使用连接数作为判断依据，需要小心多路复用的情况，如 HTTP2.0等，可能出现连接数确实是最小，当负载很高的情况；
>
> 注：P2C 算法简单理解就是随机选出两个服务节点后，如果它两都满足选取的最低阈值时，选取它两中负载最低的一个发起请求，涉及数学计算，这里不展开讲了；

&emsp;&emsp;在此基础上，通过二级缓存如 redis，解决上文提及的缓存冗余构建等问题；

## 能不能做到 鱼和熊掌可兼得？

&emsp;&emsp;既然都写出来了，那肯定是可以的！即有界负载的一致性哈希算法（详见：[Consistent Hashing with Bounded Loads](https://acceleratorssr.github.io/2024/09/17/ConsistentHashingWithBoundedLoads.html)），兼得 **哈希的优势** 和 **负载均衡的优秀表现**；

&emsp;&emsp;在此处简单说明该算法的核心，即 平衡参数（平衡因子）`C` （C > 1），而每个服务节点允许请求的接受上限（逻辑上限）为 `C` 倍的服务集群平均负载，其实如果将 `C` 设置为正无穷，此时算法就会退化为普通的一致性哈希算法，当 `C` 接近1时，算法的表现就类似于 最少连接数 的策略；

> 注：鱼和熊掌指的是：哈希的优势 和 负载均衡的优秀表现，但面对异质化的服务节点，算法原实现没有提到此处的优化，但其实只需要在算法的基础上添加权重影响服务器的负载计算即可，如处理能力差的节点会给自己的当前负载乘上一个大于1的数；
> 更细致的优化可参考本人的这篇 blog：[LB](https://acceleratorssr.github.io/2024/08/18/JD_LB.html)，介绍 JD 的方案；

补充下有界一致性哈希算法的优势：
- 不会出现服务器过载的情况；
- 当服务器不出现过载的情况时（指超出逻辑的限制，非客观过载），完美利用哈希的优势（缓存）；
- 当服务器过载时，对于相同哈希的请求，选择的回退服务器列表是相同的，即保证哈希请求第二选择的服务器都相同；
- 一样可以通过虚拟节点优化负载的分布情况；

# 结尾

&emsp;&emsp;综上，最后我在项目中也实现了负载均衡和解析器等，talk is cheap, show you code，写的不敢说完善，详见[grpc_ex](https://github.com/acceleratorssr/post/tree/master/pkg/grpc-extra)