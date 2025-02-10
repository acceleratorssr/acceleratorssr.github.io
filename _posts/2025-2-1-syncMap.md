---
layout: post
title: sync.Map
tags: 源码相关
excerpt: 使用 & 理解代码
---

# sync.Map

&emsp;&emsp;本质上是 用空间换时间，通过读写分离，减少加锁时间，适合于读多写少的场景；

&emsp;&emsp;基本结构：

```go
type Map struct {
	mu Mutex
	read atomic.Pointer[readOnly]
	dirty map[any]*entry
	misses int
}

type readOnly struct {
	m       map[any]*entry
	amended bool
}

// p 为空 代表该条目被删除
// p 等于 expunged 代表条目被删除的基础上，m.dirty 中对应条目也不存在；
type entry struct {
	p atomic.Pointer[any]
}
```

**基本逻辑结构**

1. 读层和脏层
   - 读层（read）：存储大部分的键值对，使用原子操作实现高效读访问。读层使用的是只读的数据结构，典型情况下是一个不可变的 map；
   - 脏层（dirty）：当写操作发生时，新键值对会被插入到脏层中。脏层使用一个标准的 Go map，并通过互斥锁保护；（保存 read 中所有未标记删除的数据，即存储所有 写入、删除 操作）（misses 过多时，dirty 会被提升为 read，变相缩容（移除了 dirty））
2. 数据迁移（promotion）
   - 当读取的键在读层中不存在时，可能在脏层中存在。若在脏层中找到了这个键值对，会将其提升（promote）到读层中。
3. misses 和「缓存」
   - 初次访问时，read 层为空，第一次写入会创建 dirty 层；
   - 通过缓存热数据在读层中，最大限度地减少锁的使用，提升读性能；
4. CURD 流程
   - 读：
     -  先查 read，因为是不可变结构 所以是并发安全的，无锁；
     -   如果查不到，再去 dirty 层查；
   - 写：
     - 如果此时 dirty 层不存在，说明 read 层数据是最新的，则直接拷贝 read 创建 新的 dirty 层；
     - 存在 read 则直接 CAS 更新数据（顺带更新 dirty），存在 dirty 中则直接更新；
   - 删：
     - 首先尝试在 read 中找到对应的数据，如果存在则将其标记为 expunged（是一个 any 指针，标记脏层删除的数据）；
     - 在 dirty 层尝试直接删除掉该数据；

**写逻辑分析**：

```go
func (m *Map) Store(key, value any) {
    _, _ = m.Swap(key, value)
}

func (m *Map) Swap(key, value any) (previous any, loaded bool) {
	read := m.loadReadOnly()
	if e, ok := read.m[key]; ok {
        // 尝试交换对应值的指针
		if v, ok := e.trySwap(&value); ok {
			if v == nil {
				return nil, false
			}
			return *v, true
		}
	}

    // 说明 read 没有 key 的数据，需要更新 新增数据；
	m.mu.Lock()
	read = m.loadReadOnly()
	if e, ok := read.m[key]; ok {
        // 能进来说明 上锁前 dirty 的数据被提升为 read
        // 如果此时 read 中的 key 已经被标记删除，则会基于 CAS 重新恢复
		if e.unexpungeLocked() { 
			// 并且恢复 dirty 中的数据
			m.dirty[key] = e
		}
        // 将 read 中的对应值更新
		if v := e.swapLocked(&value); v != nil {
			loaded = true
			previous = *v
		}
        // 如果 read 没有数据，dirty 有，则直接更新 dirty 中的数据
	} else if e, ok := m.dirty[key]; ok {
		if v := e.swapLocked(&value); v != nil {
			loaded = true
			previous = *v
		}
        // read、dirty 都没有数据
        // 给 read 打上 amended 标记（标记意为 dirty 有 read 没有存的数据）
        // 更新 dirty 即可
	} else {
		if !read.amended {
            // 创建 dirty
			m.dirtyLocked()
			m.read.Store(&readOnly{m: read.m, amended: true})
		}
		m.dirty[key] = newEntry(value)
	}
	m.mu.Unlock()
	return previous, loaded
}

func (e *entry) trySwap(i *any) (*any, bool) {
	for {
		p := e.p.Load()
        // 即 dirty 已经删除了这个数据（dirty 层本身存在，但是槽位不存在）
        // read 还有（被标记为已删除）
        // 此时视为不需要更新当前数据?标记为删除再写入？
		if p == expunged {
			return nil, false
		}
        // 即 dirty/read 都有数据
		if e.p.CompareAndSwap(p, i) {
			return p, true
		}
	}
}
```

**读逻辑分析**：

```go
func (m *Map) Load(key any) (value any, ok bool) {
	read := m.loadReadOnly()
	e, ok := read.m[key]
    // 如果 read 中没数据，且有标记代表 dirty 有对应 key
	if !ok && read.amended {
		m.mu.Lock()
		// 防 dirty 被提升为 read
		read = m.loadReadOnly()
		e, ok = read.m[key]
		if !ok && read.amended {
			e, ok = m.dirty[key]
			// Regardless of whether the entry was present, record a miss: this key
			// will take the slow path until the dirty map is promoted to the read
			// map.
			m.missLocked() // misses ++
		}
		m.mu.Unlock()
	}
	if !ok {
		return nil, false
	}
	return e.load()
}

func (m *Map) missLocked() {
	m.misses++
	if m.misses < len(m.dirty) {
		return
	}

    // dirty 提升为 read
	m.read.Store(&readOnly{m: m.dirty})
	m.dirty = nil
	m.misses = 0
}
```

读性能优化策略：
1. 读层的无锁操作
   - 读层是只读的，访问不需要加锁，通过原子操作实现无锁访问，大大提升了并发读的性能。
2. 数据提升机制
   - 当数据存在于脏层中时，会将其提升到读层，这样后续对该数据的读取操作就可以直接在读层完成，避免频繁访问脏层；
3. 读写分离
   - 通过将读操作和写操作分离，在读多的情况下保持高效，读操作大多数情况下不需要加锁，而写操作使用 CAS、互斥锁 保护 read/dirty；

**删逻辑分析**：

```go
  func (m *Map) LoadAndDelete(key any) (value any, loaded bool) {
	read := m.loadReadOnly()
	e, ok := read.m[key]
	if !ok && read.amended {
		m.mu.Lock()
		read = m.loadReadOnly()
		e, ok = read.m[key]
		if !ok && read.amended {
			e, ok = m.dirty[key]
			delete(m.dirty, key)
			m.missLocked() // misses ++
		}
		m.mu.Unlock()
	}
	if ok {
		return e.delete()
	}
	return nil, false
}

// 当 key 存在于 read 中时
func (e *entry) delete() (value any, ok bool) {
	for {
		p := e.p.Load()
        // 判断当前是否已经为空或者标记为删除，直接返回删除失败
		if p == nil || p == expunged {
			return nil, false
		}
        // CAS 将 p 置为 nil
		if e.p.CompareAndSwap(p, nil) {
			return *p, true
		}
	}
}
```