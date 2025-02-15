---
layout: post
title: Consistent Hashing with Bounded Loads
tags: 分布式
excerpt: 有界一致性哈希
---

# start
&emsp;&emsp;核心目的是，希望将一组客户端（balls）均匀分配给一组服务器（bins），从而最小化所有服务器的最大负载，并最小化动态拓扑变化时，数据移动的次数；

&emsp;&emsp;完全动态分配问题的标准解决方案：一致性哈希 [SML+03， KLL+97]，虽然最大限度减少了预期的移动次数，但是也可能导致服务器严重过载，负载均衡方面甚至不比客户端随机分配给服务器要好；

&emsp;&emsp;至于为什么会过载，举例而言：m个球（数据），n个桶（节点），期望负载为 $\dfrac{m}{n}$；

&emsp;&emsp;将球和桶哈希到一个单位圆上时，每个球顺时针分配到最近的桶里，根据概率学分析，这种区间的最大长度可以达到 $\Theta\left(\dfrac{m \log n}{n}\right)
$，即桶的最大负载会比平均值高出 $\Theta(\log n)
$ 倍；

&emsp;&emsp;而如果针对球分析，桶之间的期望距离是 $(\dfrac{1}{n})$，将一个球也算入后，它与最近桶的期望距离即为 $(\dfrac{1}{n+1})$，所以当哈希值处于 $(\dfrac{2 \times 1}{n+1})$ 区间中的球，会被分配进一个桶内，即期望落入该区间的球数量为 $\dfrac{2m}{n+1} \approx \dfrac{2m}{n}$；此时负载几乎为平均值的两倍；

### 核心思想
&emsp;&emsp;通过转发机制解决负载均衡问题，为每个桶设定一个容量限制，超过容量限制的负载会被转发到其他桶内；设负载均衡的**平衡参数** c > 1，确保没有一个桶的容量大于 $ \left\lceil \dfrac{cm}{n} \right\rceil $；

&emsp;&emsp;平衡参数 $ c = 1 + ϵ $ 是用来控制桶在分配球时的负载上限。参数 $ c $ 决定了每个服务器的最大负载，而 $ ϵ $ 控制了负载均衡的松弛度；

即：

&emsp;&emsp;平衡参数 $ c $ 增大时，意味着服务器的负载允许更大的浮动和松弛，负载上限为$ \left\lceil \dfrac{cm}{n} \right\rceil $，

&emsp;&emsp;$ ϵ ≤ 1 $ 时， $ c $ 接近 1，适用于负载较低、需要更严格的均衡场景；系统会更加严格地保证服务器负载不会超出平均负载太多，即更多的客户端请求迁移以保证负载均衡；此时迁移的客户端数量的期望值是 $O\left(\dfrac{1}{\epsilon^2}\right)$，即负载越小（紧绷），迁移量越大；

&emsp;&emsp;$ ϵ > 1 $ 时， $ c $ 大于 2，适用于负载较高且可以容忍更大波动的场景；系统允许服务器的负载有更多的松弛，减少了在新增或删除服务器时客户端的迁移量；期望变为 $O\left(\dfrac{\log c}{c}\right)$，即负载越大（松弛），迁移量越小；

 
#### 参考资料：
[Consistent Hashing with Bounded Loads(paper)](https://arxiv.org/abs/1608.01350)
[Consistent Hashing with Bounded Loads(Google Research)](https://research.google/blog/consistent-hashing-with-bounded-loads/)


[SML+03]Ion Stoica, Robert Morris, David Liben-Nowell, David R. Karger, M. Frans Kaashoek, Frank Dabek, and Hari Balakrishnan.Chord: a scalable peer-to-peer lookup protocol for internet applications.IEEE/ACM Trans. Netw., 11(1):17–32, 2003.

[KLL+97]David R. Karger, Eric Lehman, Frank Thomson Leighton, Rina Panigrahy, Matthew S. Levine, and Daniel Lewin.Consistent hashing and random trees: Distributed caching protocols for relieving hot spots on the world wide web.In Proceedings of the Twenty-Ninth Annual ACM Symposium on the Theory of Computing, El Paso, Texas, USA, May 4-6, 1997, pages 654–663, 1997.