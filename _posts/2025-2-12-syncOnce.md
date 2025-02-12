---
layout: post
title: sync.Map
tags: 源码相关
excerpt: 使用 & 理解代码
---

> init：只会被执行一次，当所属 package 被 import时，会执行当前同一 package 下的 init 函数，执行顺序：
> - 单文件内按照声明的顺序；
> - 多文件按照文件名的字典序；
> - 如果当前 package 有依赖的其他包，会先执行其他包的 init 函数 以此类推；
>
> import 的 package 包初始化过程：常量 -> 全局变量 -> init()

sync.Once 用于在指定时机只执行一次指定的方法；

# sync.Once

**sync.Once 并不提供对执行结果的判断能力**；


```go
// 1.23.2
type Once struct {
	done atomic.Uint32
	m    Mutex
}
func (o *Once) Do(f func()) {
	if o.done.Load() == 0 {
		o.doSlow(f)
	}
}

func (o *Once) doSlow(f func()) {
	o.m.Lock()
	defer o.m.Unlock()
	if o.done.Load() == 0 {
		defer o.done.Store(1)
		f()
	}
}

```

&emsp;&emsp;通过mutex控制同一时间只有一个goroutine执行任务；

&emsp;&emsp;**双重检测锁机制**（Double-checked Locking），用于解决**高并发场景下 情性创建对象、实例**，在本处是惰性执行任务（在第一次判断任务未执行 到 上锁 的时间 diff 中，可能阻塞多个 goroutine,需要二次判断防止重复执行）；

&emsp;&emsp;双 defer 用于保证任务执行完成后能先标记 已执行，再释放绩；

**核心逻辑**：
1. 检查任务是否已经执行过(atomic.LoadUint32(&o.done))
2. 加质，将解销提作压楼：
3. 二次判断任务是否没有完成执行；
4. 将标记 任务已完成 操作压栈；
5. 执行任务；
6. 先标记任务已完成后，解锁；

**注意**：
- 如果在10 中重复调用 同一Once的 Do，会导致死锁，如：
```go
var once sync.Once

func recursive(){
    once.Do(func(){
        recursive()
    })
}
```

Q：为什么不使用乐观锁实现检查任务完成否，即CAS操作？

A：核心在于 Do 语义是保证返回时，方法一定已经被执行过一次了，即使可能不是当前返回的 goroutine 执行的；而如果使用 CAS 则会导致其他 CAS 失败的 goroutine 提前返回，和预期不符，即返回时方法可能尚未被执行完成；