---
layout: post
title: Gorm callback相关细节
tags: go
excerpt: 
---

> Callbacks are registered at the global *gorm.DB level, not on a session basis. This means if you need different callback behaviors, you should initialize a separate *gorm.DB instance.

回调是基于全局`*gorm.DB`，而不是在会话级别;

global `*gorm.DB`:

`db.callbacks`
- 全局的回调管理器，负责管理所有的回调，能够访问所有注册的回调，并且可以在执行任何数据库操作时统一管理回调的执行；
- 允许在此注册全局回调，（默认六种操作，crud和原生sql和单行查询，没找到添加第七种操作的方法，看上去不得行；-不过貌似也没必要-）

`processor.callbacks`
- 回调处理器（processor）的属性，六种操作的回调列表，保存callback；
- 默认没有回调，初始化 processors 时（initializeCallbacks），六种操作的默认值只有对应的 db；

每个数据库的操作对应一个processor，而每个processor会管理一组与操作相关的回调函数；

```go
// callbacks gorm callbacks manager
type callbacks struct {
	processors map[string]*processor
}

type processor struct {
	db        *DB
	Clauses   []string
	fns       []func(*DB)
	callbacks []*callback
}

type callback struct {
	name      string
	before    string
	after     string
	remove    bool
	replace   bool
	match     func(*DB) bool
	handler   func(*DB)
	processor *processor
}

func initializeCallbacks(db *DB) *callbacks {
	return &callbacks{
		processors: map[string]*processor{
			"create": {db: db},
			"query":  {db: db},
			"update": {db: db},
			"delete": {db: db},
			"row":    {db: db},
			"raw":    {db: db},
		},
	}
}
```

### 注册链路（其他操作同理）：
```go
// 如果before注册为*，则优先级最高，放前面，（after就反过来）
// 如果为空，则代表无所谓执行顺序，即 可独立执行，无依赖；
// 可以填入其他回调的name（此处name为prometheus_query_before），保证在其之前执行
err = db.Callback().Query().Before("*").
	Register("prometheus_query_before", promCB.before())

// Callback returns callback manager
func (db *DB) Callback() *callbacks {
    return db.callbacks
}

func (cs *callbacks) Update() *processor {
	return cs.processors["update"]
}

func (p *processor) Before(name string) *callback {
	return &callback{before: name, processor: p}
}

func (c *callback) Register(name string, fn func(*DB)) error {
	c.name = name
	c.handler = fn
	c.processor.callbacks = append(c.processor.callbacks, c)
    // 将callback的handler排序后写入processor的fns，执行时遍历调用
	return c.processor.compile() 
}

func (p *processor) compile() (err error) {
	var callbacks []*callback
	removedMap := map[string]bool{}
	for _, callback := range p.callbacks {
        // handler 或者 返回true的match 视为有效的回调方法
		if callback.match == nil || callback.match(p.db) { 
			callbacks = append(callbacks, callback)
		}
		if callback.remove {
			removedMap[callback.name] = true
		}
	}

	if len(removedMap) > 0 {
		callbacks = removeCallbacks(callbacks, removedMap)
	}
	p.callbacks = callbacks
    
    // 上面整理好回调状态后，进行排序，注意此处可能有重复的回调
	if p.fns, err = sortCallbacks(p.callbacks); err != nil {
		p.db.Logger.Error(context.Background(), "Got error when compile callbacks, got %v", err)
	}
	return
}

func sortCallbacks(cs []*callback) (fns []func(*DB), err error) {
	var (
		names, sorted []string
		sortCallback  func(*callback) error
	)
    // 把 before 和 after 为允许所有类型sql，即*的回调放前面
	sort.SliceStable(cs, func(i, j int) bool {
		if cs[j].before == "*" && cs[i].before != "*" {
			return true
		}
		if cs[j].after == "*" && cs[i].after != "*" {
			return true
		}
		return false
	})

    ...

    // 执行排序，并干掉remove回调
	for _, c := range cs {
		if err = sortCallback(c); err != nil {
			return
		}
	}

	for _, name := range sorted {
		if idx := getRIndex(names, name); !cs[idx].remove {
			fns = append(fns, cs[idx].handler)
		}
	}

	return
}
```

#### sortCallbacks中的核心排序方法:
**注意：实际上并没有区分 before 和 after，因为这只是处理所有回调的注册和排序；**
```go
// names内含所有 存活 回调函数
for _, c := range cs {
    // show warning message the callback name already exists
    if idx := getRIndex(names, c.name); idx > -1 && !c.replace && !c.remove && !cs[idx].remove {
        c.processor.db.Logger.Warn(context.Background(), "duplicated callback `%s` from %s\n", c.name, utils.FileWithLineNum())
    }
    names = append(names, c.name)
}

// getRIndex get right index from string slice

sortCallback = func(c *callback) error {
    if c.before != "" { // if defined before callback
        if c.before == "*" && len(sorted) > 0 { // *优先级最高
            if curIdx := getRIndex(sorted, c.name); curIdx == -1 {
                sorted = append([]string{c.name}, sorted...)
            }
            // 当前回调注册时，要求在 before 回调前执行，进入该分支则代表已存在 before 回调
        } else if sortedIdx := getRIndex(sorted, c.before); sortedIdx != -1 {
            // 如果当前回调不存在，则正常塞到 before 前即可
            if curIdx := getRIndex(sorted, c.name); curIdx == -1 {
                // if before callback already sorted, append current callback just after it
                sorted = append(sorted[:sortedIdx], append([]string{c.name}, sorted[sortedIdx:]...)...)
            } else if curIdx > sortedIdx { // 顺序冲突，逻辑的顺序又前又后导致的
                return fmt.Errorf("conflicting callback %s with before %s", c.name, c.before)
            }
            // 当前回调还没有被排序，则默认将它添加到 sorted 的最后
        } else if idx := getRIndex(names, c.before); idx != -1 {
            // if before callback exists
            cs[idx].after = c.name
        }
    }

    if c.after != "" { // if defined after callback
        if c.after == "*" && len(sorted) > 0 {
            if curIdx := getRIndex(sorted, c.name); curIdx == -1 {
                sorted = append(sorted, c.name)
            }
        } else if sortedIdx := getRIndex(sorted, c.after); sortedIdx != -1 {
            if curIdx := getRIndex(sorted, c.name); curIdx == -1 {
                // if after callback sorted, append current callback to last
                sorted = append(sorted, c.name)
            } else if curIdx < sortedIdx {
                return fmt.Errorf("conflicting callback %s with before %s", c.name, c.after)
            }
            // after 需要确保指定的回调（即 c.after）已经在 sorted 切片中排好序，
            // 否则必须先处理 after 回调，再处理当前回调
            // 存在 递归依赖
        } else if idx := getRIndex(names, c.after); idx != -1 {
            after := cs[idx]

            if after.before == "" {
                after.before = c.name
            }

            if err := sortCallback(after); err != nil {
                return err
            }

            if err := sortCallback(c); err != nil {
                return err
            }
        }
    }

    // if current callback haven't been sorted, append it to last
    if getRIndex(sorted, c.name) == -1 {
        sorted = append(sorted, c.name)
    }

    return nil
}
```

在执行主操作之前（after 同理），GORM 会检查并执行与该操作相关的 before 回调：
- 查找回调: GORM 会从处理器（processor）的回调列表（callbacks）中查找所有注册的 before 回调；
- 执行回调: 按照注册顺序依次调用这些回调函数；
执行链路（以first为例）：

```go
func (db *DB) First(dest interface{}, conds ...interface{}) (tx *DB) {
	...
	return tx.callbacks.Query().Execute(tx) // 获取query的处理器并调用执行方法
}

func (p *processor) Execute(db *DB) *DB {
    ...
	for _, f := range p.fns { //执行回调
		f(db)
	}
    ...
}
```

### 应用
通过实现 Plugin 接口，可以应用自定义回调；
```go
type Plugin interface {
  Name() string
  Initialize(*gorm.DB) error
}

db.Use(NewCallbacks())
```

