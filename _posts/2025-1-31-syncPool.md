---
layout: post
title: sync.Pool & ringBuffer
tags: 源码相关
excerpt: 使用 & 理解代码
---

[代码实践]()

## pool

&emsp;&emsp;核心思想就是每个 P 都有私有池、共享池，G 存取对象前 先绑定一个 P 防止竞争，先从 私有池 中获取（无锁），再去 共享池 中获取（首对象，加锁），如果共享池为空，去其他 P 上偷一个（尾元素，加锁），再没有，去受害者缓存中拿一个（同），最后兜底是创建一个新对象并返回；

> &emsp;&emsp;受害者缓存（**victim 机制**）：**个人理解**是指：每轮 GC 处理 sync.Pool 时，会将没有强引用的对象移动到 victim cache 中，当然在此之前会先清空上一轮 GC 移动的对象（队列形式组织）；
> &emsp;&emsp;在 Get 过程中，如果遍历发现获取对象失败（priviate 和 shared 都为空），则清空 victimSize 为0，告知其他 goroutine 不需要再查询 victim cache ；（注：此时没有立刻 GC 掉 victim）

```go
type poolLocalInternal struct {
	private any       // 仅可被当前 P 使用
	shared  poolChain // 当前 P 可 pushHead/popHead; 其他 P 可 popTail（偷）.
}
```

**结构**：

```go
type Pool struct {
	noCopy noCopy

	local     unsafe.Pointer    // 每个 processor 的本地队列，存储固定大小的对象池；实际类型是 [P]poolLocal
	localSize uintptr           // len(poolLocal)

	victim     unsafe.Pointer   // 上一轮 GC 后剩余的对象，可被重用
	victimSize uintptr          // len(victim)

    // 构造函数
    // 注意使用 Get 时不能修改 New，存在并发问题
	New func() any
}
```

<br>

**方法**：
`Put(x any)`: 将 x 放入池中，等待被 Get() 取出使用；

```go 
func (p *Pool) Put(x any) {
	if x == nil { // 不存 nil
		return
	}

    // 用于 debug 时测试并发问题
	if race.Enabled {
		...
	}

    // 将当前 goroutine 绑定到一个 P 上，返回获取 poolLocal
    // 防 data-race
	l, _ := p.pin()

	if l.private == nil { // 当前 P 无缓存对象
		l.private = x
	} else {
		l.shared.pushHead(x)
	}

	runtime_procUnpin()

	...
}
```

`Get() any`: 从对象池中获取一个对象，如果对象池为空，则创建一个新对象(`New func() any`)并返回；

```go
func (p *Pool) Get() any {
	...
	l, pid := p.pin()

	x := l.private // O(1) 获取时间复杂度

	l.private = nil

	if x == nil {
		x, _ = l.shared.popHead() // 时间局部性，理解为使用刚被推进来的最新对象
		if x == nil {
			x = p.getSlow(pid) // 注1
		}
	}

	runtime_procUnpin()

	...

	if x == nil && p.New != nil {
		x = p.New() // 兜底
	}
	return x
}
```

> 注1：先 从其他 goroutine 下的本地池（共享池）中偷取对象（尾部偷），如果没有，再去 受害者缓存（victim cache 中尝试获取）

#### 实践策略 小 总结（仅为推荐 也可能在具体的场景下有更好的策略）

1、设置 New 函数，防止一开始使用 空pool 导致获取对象失败；

2、并发安全，无需加锁；

3、使用完成后尽快归还（put），加快复用效率；

4、复用的对象应该是短期内频繁使用的且小，不应是长期持有的（如 mysql 连接）或者过大；

## ringBuffer
poolChain：

```go
type poolDequeue struct {
	headTail atomic.Uint64 // 高位是写指针，低是读指针

	vals []eface
}
```

&emsp;&emsp;就和正常的环形数组一样，指针会取模遍历；

> **注**：一般数组长为 2^n，此时 `i % (2^n) == i & ((2^n)-1)`，一般情况下位操作是最快的；
> 
> 推导：`&(2^n)-1`代表保留数值的低n位，而取模计算获取的余数也是数值的低n位（n+1位就是一个单位，为2^n，被除掉了）；

&emsp;&emsp;双指针（读、写指针），读指针到写指针的区间为缓存数据，支持一个线程写，

pushHead 流程是先占用当前槽位，再移动 headTail，

```go
func (d *poolDequeue) pushHead(val any) bool {
	ptrs := d.headTail.Load()
	head, tail := d.unpack(ptrs)
	if (tail+uint32(len(d.vals)))&(1<<dequeueBits-1) == head {
		// Queue is full.
		return false
	}
	slot := &d.vals[head&uint32(len(d.vals)-1)]

	// Check if the head slot has been released by popTail.
	typ := atomic.LoadPointer(&slot.typ)
	if typ != nil {
		// Another goroutine is still cleaning up the tail, so
		// the queue is actually still full.
		return false
	}

	// The head slot is free, so we own it.
	if val == nil {
		val = dequeueNil(nil)
	}
	*(*any)(unsafe.Pointer(slot)) = val

	// Increment head. This passes ownership of slot to popTail
	// and acts as a store barrier for writing the slot.
	d.headTail.Add(1 << dequeueBits)
	return true
}

```

&emsp;&emsp;popTail 流程其实可以看作读请求（pophead 同理），基于 CAS 移动读指针的位置，只有移动成功的 goroutine 才会获取数据并退出 CAS 的重试循环，注意，此处抢占的值应该是0有效，即获取数据同时自动抢占成功，最后释放时是将标志位置为 nil；

```go
func (d *poolDequeue) popTail() (any, bool) {
	var slot *eface
	for {
		ptrs := d.headTail.Load()
		head, tail := d.unpack(ptrs)
		if tail == head {
			// Queue is empty.
			return nil, false
		}

		// Confirm head and tail (for our speculative check
		// above) and increment tail. If this succeeds, then
		// we own the slot at tail.
		ptrs2 := d.pack(head, tail+1)
		if d.headTail.CompareAndSwap(ptrs, ptrs2) {
			// Success.
			slot = &d.vals[tail&uint32(len(d.vals)-1)]
			break
		}
	}

	// We now own slot.
	val := *(*any)(unsafe.Pointer(slot))
	if val == dequeueNil(nil) {
		val = nil
	}

	// Tell pushHead that we're done with this slot. Zeroing the
	// slot is also important so we don't leave behind references
	// that could keep this object live longer than necessary.
	//
	// We write to val first and then publish that we're done with
	// this slot by atomically writing to typ.
	slot.val = nil
	atomic.StorePointer(&slot.typ, nil)
	// At this point pushHead owns the slot.

	return val, true
}
```

当一个 ringBuffer 满了之后，会新申请一个2倍大小的 ringBuffer，以双向链表的形式组织；

> sync.Pool 中的对象算是弱引用？因为多一段缓存的存活期；

