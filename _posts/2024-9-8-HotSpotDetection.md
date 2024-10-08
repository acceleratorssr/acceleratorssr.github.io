---
layout: post
title: 热点监测
tags: 技术沉淀
excerpt: 实时监测热点的相关算法
---

# start

&emsp;&emsp;二八定律：头部20%的数据占据80%的访问流量；

&emsp;&emsp;但热点无法预测，只能监测；

预判的方案基本分为：

&emsp;&emsp;小数据量情况下，应用实例在本地缓存全部信息，定时推送更新，缺点是不同业务需要实现各自的local cache相关逻辑；

&emsp;&emsp;通过计数判断热点，即为上一时刻的热点提供本地缓存，同样需要实现不同业务各自的逻辑；

### 热点检测与治理框架：
热点检测+本地缓存；
- 实现监控热点；
- 识别突发热点，并字段将其放入本地缓存；
- 高效本地缓存策略，提供缓存命中率；
- 支持提前将数据添加到本地缓存（白名单）；
- 通过配置开启热点检测与本地缓存，非侵入式接入；

#### 热点检测算法（用于数据流中的频繁元素检测的算法，topk大象流识别）：
##### 1、Lossy Counting（admit-all-count-some）

**核心流程**：

&emsp;&emsp;将数据流切分成大小一致的窗口（size取决于期望错误率）；

&emsp;&emsp;分别统计每个窗口中元素出现的频率，切换计算下一个窗口时，将所有元素的频率-1，并且删除计数值为0的元素，直到到达数据检查点或者窗口处理完成，排序内存中的剩余元素得到top k；

**核心理念**：

&emsp;&emsp;高频元素遵循 长尾分布（也叫帕累托分布）。即，少数元素频率非常高，占据了总频率的较大部分，而大量低频元素几乎可以忽略不计，

&emsp;&emsp;通过引入一个错误边界 ε ，有意识地进行近似计算，允许一定范围的误差。这个错误率是通过算法[1]中定义的桶和窗口大小来控制的。错误率的上限是可以事先设定的，控制在一个可接受的、预先定义的阈值内；

##### 2、Count-Min Sketch
**核心流程**：

&emsp;&emsp;通过多个哈希函数，将元素映射到多个计数器数组中，可将其看为一个二维计数器数组，d行w列，有d个独立的哈希函数，w个计数器[2]，映射关系为<code>idx = h(n) % w</code>；在处理数据流时，通过最小堆维护 **topk**，对于每个元素，频率估计值取其 d个哈希 后得到的 多个计数器 的最小值，<code>f(x) = min<sub>1≤i≤d</sub>C[i][h<sub>i</sub>(x)]</code>；

**核心理念**：

&emsp;&emsp;数学上可证明多增加一行（一个哈希函数），会使bad estimate（错误估计）的几率减半，且长度为 w 的sketch，错误率与 w 成反比；通常 w 设为 ⌈e<sup>ϵ</sup>⌉ ，ϵ 是允许的误差范围，<code>ϵ =(Cmax−Cest) / Cest</code>，Cest为估计值， Cmax为实际计数值；

##### 3、HeavyKeeper
**核心流程**：

&emsp;&emsp;在CM sketch的基础上，在每一个bucket里，额外存储哈希指纹，遇到哈希冲突时不再是+1，而是将计数值进行衰减，衰减概率：<code>P<sub>decay</sub> = b<sup>-C</sup></code> (b > 1)；（b为衰减因子，C为计数值）

**核心理念**：

&emsp;&emsp;计数值增大时，衰减概率降低，大象流可保持稳定，淘汰鼠流；


### 参考文章：
[1] [Lossy Counting频数估计算法](https://shixiaogang.com/sketches/lossy-counting/)

[2] [Data Sketching笔记](https://longaspire.github.io/blog/Data%20Sketching%E7%AC%94%E8%AE%B0/#Count-Min-Sketch)

[3] [Count-Min Sketch](https://zhuanlan.zhihu.com/p/84688298)

[4] [论文阅读笔记：HeavyKeeper: An Accurate Algorithm for Finding Top-k Elephant Flows](https://zhuanlan.zhihu.com/p/653797371)

[参考文章：热点检测治理](https://mp.weixin.qq.com/s/C8CI-1DDiQ4BC_LaMaeDBg)