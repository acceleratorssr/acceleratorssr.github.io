---
layout: post
title: 分布式锁
tags: 分布式
excerpt: 简单聊聊分布式锁；
---

## 基于Redis如何实现一个分布式锁？它足够安全吗？
&emsp;&emsp;简单加解锁就是SETNX -> 操作共享资源 -> DEL；

&emsp;&emsp;严谨的过程： SET $lock_key $unique_id EX $expire_time NX

&emsp;&emsp;->    &emsp;&emsp;定时给锁续约；

&emsp;&emsp;->    &emsp;&emsp;使用Lua脚本，执行get判断是否可以del；

1. 问题出现：**死锁**；

&emsp;&emsp;如果程序处理业务逻辑异常，或者直接宕机，锁没有被及时释放，会导致死锁；

&emsp;&emsp;**逻辑异常**的解决方法：<code>defer redis.del(key)</code>

&emsp;&emsp;其他语言就是做异常捕获：<code>try ... catch ... fianlly redis.del(key)</code>

&emsp;&emsp;**宕机**的解决方法：给锁设置一个租期，等于一个超时时间；
> 注意：在Redis 2.6.12版本前加锁和设置租期是两个原子命令，中间可能会出现宕机等问题导致租期设置失败，故在版本后加锁和设租期变成一个原子指令：SET lock 1 EX 10 NX(NX 表示只有在键不存在时才设置成功)；

2. 问题出现：锁过期后，**释放其他节点持有的锁**；

&emsp;&emsp;锁过期的解决方法：加锁后开启**守护线程**(也可以说goroutine)，守护线程负责定时续约，以免锁被释放；
> 守护进程/线程：后台长时间/周期性执行任务，且不影响前台，通常伴随程序的启动/终止而创建/停止；

&emsp;&emsp;释放其他节点的锁：

&emsp;&emsp;加锁时设置锁的归属（唯一标识）（如：设置锁的value为节点的UUID，线程id），解锁时检查锁是否还属于自己；

&emsp;&emsp;通常实现是先get对比唯一标识后，判断是否del，但此处任然是两条原子性命令，中间可能发生问题；

&emsp;&emsp;**解决方法**：将get和del一起放入Lua脚本中执行，因为redis处理客户端请求是单线程的，所以在此期间不会被打断；
```lua
if redis.call("GET", KEYS[1]) == ARGV[1]
then
  return redis.call("DEL", KEYS[1])
else
  return 0
end
```

## redis的主从对分布式锁的影响？
**问题**：由于主从同步是**异步**复制，当master加锁成功后直接宕机了，而slave还没复制得到该lock key，HA（高可用性系统）切主（哨兵负责监控），slave被提升为new master后，锁就丢失了；

**解决方案**：redlock（多台运行redis实例的机器的时钟不能误差过大，否则锁会不正常释放，损坏大多数的正确性）：
1、 客户端先获取当前时间戳T1；

<br>

2、 客户端依次向这5个Redis实例发起加锁请求（SET），且每个请求会设置超时时间（ms级别，远小于锁的有效时间），如果某一实例加锁失败（包括网络超时、锁已经被其他客户端获取等情况），就立刻向下一个Redis实例申请加锁；
> 逐一请求的原因:避免竞争：依次请求可以减少在多个Redis实例之间的竞争情况。假如所有请求同时发起，可能会引起资源争夺，导致某些请求失败或者延迟增加；

<br>

3、 如果客户端得到大多数Redis实例响应加锁成功，则再次获取当前时间戳T2，如果T2-T1 < 锁的过期时间，此时认为客户端加锁成功；

<br>

4、 如果成功则正常操作，失败则向全部节点（redis实例可能已经写入lock key了，但是返回过程中网络原因丢了或者超时，所以要保证所有redis释放锁）发送释放锁请求（Lua脚本）；

## 一个严谨的分布式锁模型如何实现，应该考虑的问题
1、 分布式锁的目的：

- 效率：避免执行同一个任务；（也需要幂等性，即多执行一次也不能出问题）
- 正确性：操作敏感的共享资源；（重复操作会导致数据错误）

2、 锁在分布式系统中会遇到的NPC问题：

- <code>N</code>：Network Delay，网络延迟；
- <code>P</code>：Process Pause，进程暂停（比如：执行GC，等待IO？）（前提是客户端拿锁后发生）；
  - 进程暂停导致锁过期，如果立刻其他客户端申请锁成功后，原进程恢复后的操作就发生数据竞争了（redis删掉锁后不会通知客户端）；
- <code>C</code>：Clock Drift，时钟飘逸；
  - 即多台机器运行redis实例，如果其中一台机器的时钟发生跳跃，导致该redis实例的锁过期，此时集群就丧失了大多数的正确性（即多个客户端可同时持有锁）；
  - 调整时钟的方法：使用NTP服务平滑调整时钟，可避免跳跃；

<br>

&emsp;&emsp;一种可能的解决方法：在客户端于redis实例正常加锁交互的过程中，多加一个token递增字段（或者直接用lock key的uuid值作比较），redis实例可以依次判断请求是否过时；

&emsp;&emsp;**效率**：不使用redlock，直接主从加哨兵（参考8~10w的qps）；

&emsp;&emsp;**正确性**：使用主从加哨兵遇到的失效问题：上层使用Redis锁过滤并发（性能高）加资源层版本号（像是CAS？）（类似mysql的乐观锁，ZK、etcd的Revision）；

> 利用mysql的事务也能实现分布式锁；

## Redis、Zookeeper、Etcd的分布式锁间优劣？
#### ZK分布式锁
&emsp;&emsp;是通过创建一个临时节点的方式；即创建临时节点 -> 操作共享资源 -> 删除临时节点；
&emsp;&emsp;保证客户端持有锁的机制不是设置过期时间，而是让客户端定时发送心跳包，当客户端没有维持心跳时，ZK就会删掉这个临时节点；
> 注意：客户端拿到锁后，发生进程暂停的情况下，锁如上所说依然会失效；

&emsp;&emsp;ZK的优势在于，基于watch机制，可以实现公平锁，即每次不同客户端加锁的请求都创建为临时节点，都挂载于lock节点下，节点按照次序排队，如果节点发现在当前lock节点下有比自己更小的节点，就对前一个节点进行监视，否则就正常获取锁，操作；

&emsp;&emsp;简单来说：加锁失败后不立刻返回结果到客户端，而是利用watch机制等待前一个节点（此处的前一个节点为上一把锁）的释放，当前一个节点释放时，就立刻加锁并处理资源；
> watch机制：监听某一个节点的任何变化，同时仅会触发一次通知；
公平锁：当多个线程同时请求锁时，锁的获取应该按照它们请求的先后顺序进行；

#### Etcd分布式锁
&emsp;&emsp;创建租约（设置过期时间） -> 创建节点（携带这个租约） -> 定时续约（保持锁） -> 删除节点（释放锁）
注意：客户端拿到锁后，发生进程暂停的情况下，锁如上所说依然会失效；

&emsp;&emsp;同样有watch机制实现公平锁；