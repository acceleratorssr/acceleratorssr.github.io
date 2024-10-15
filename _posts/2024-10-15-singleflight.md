---
layout: post
title: singleflight
tags: 源码相关
excerpt: 解决单个进程内的高并发请求问题;
---

# 前情提要

> `runtime.Goexit` 终止调用它的 goroutine，其他 goroutine 不受影响，终止前会执行 defer，不是一种 panic，`recover` 只会返回空；
> main goroutine 调用 `runtime.Goexit` 后，不会终止其他 goroutine 的执行，当所有 goroutine 退出后，程序会崩溃；

**eg**:
输出：
`1 2 3 4 No panic 5 i am here`
`fatal error: no goroutines (main called runtime.Goexit) - deadlock!`
```go
func main() {
	defer func() {
		fmt.Printf("5 ")
	}()
	fmt.Printf("1 ")

	go func() {
		defer func() {
			fmt.Printf("4 ")
			if r := recover(); r != nil {
				fmt.Printf("Get it ")
			} else {
				fmt.Printf("No panic ")
			}
		}()
		fmt.Printf("3 ")
		runtime.Goexit()
		fmt.Printf("xxxxxx ")
	}()

	fmt.Printf("2 ")
	time.Sleep(time.Second)

	go func() {
		time.Sleep(time.Second)
		fmt.Printf("i am here\n")
	}()
	runtime.Goexit()
}

```

# 正餐
&emsp;&emsp;在`singlefilght`中，**作用是在任意时刻仅允许一个 `goroutine` 执行用户指定的函数，允许多个 goroutine 以同步或者异步的方式接收唯一的执行结果**，并且也会接收到执行的 err，和一个 是否有其他 goroutine 共享该结果 的标记

&emsp;&emsp;核心使用就是 `Group` 结构体，内部维护一个互斥锁和一个map，map 用于维护用户传入的 key 及其对应的执行器（`call`）的指针，并且采用懒加载的策略，当第一次调用 `Do` 或者 `DoChan` 方法才会初始化 map 内存；
```go
type Group struct {
	mu sync.Mutex
	m  map[string]*call
}
```

&emsp;&emsp;执行器主要包含了一个 `waitgroup`、存储结构的 `val`、注册需要异步返回结果的 请求数量 `dups`、用于异步返回结果的 `channel`；
```go
type call struct {
	wg sync.WaitGroup

	val interface{}
	err error

	dups  int
	chans []chan<- Result
}
```

&emsp;&emsp;`Group` 的执行函数有 `Do` 和 `DoChan` 两种形式，分别代表 goroutine 期望同步返回结果，还是异步，它两的核心流程区别就在非第一个请求的 goroutine 进入后，前者利用 `waitgroup` 直接阻塞等待结果（以 key 为粒度），后者将自己接收结果的 channel 注册给当前执行器后，直接返回该 channel；

&emsp;&emsp;他们两都是通过 `doCall` 方法执行用户传入的执行函数 `fn`，不考虑容错的前提下，其实这个方法就是单纯执行 `fn` 后，将结果和 err 放入执行器内，调用 `wg` 的 `Done` 并删除当前注册的 key 后返回；此时对应其他 goroutine，也会收到该结果；

&emsp;&emsp;在 `doCall` 内的容错实现中，目的需要区分 `panic` 和 `runtime.Goexit()` 这两种错误，比较巧妙地利用了两个标志和两个 defer：

```go
func (g *Group) doCall(c *call, key string, fn func() (interface{}, error)) {
	normalReturn := false
	recovered := false

	defer func() {
        // 因为 runtime.Goexit() 无法被捕获
        // 所以 recovered 为 false 时，一定发生了 runtime.Goexit()
        // 注：!normalReturn 表明没有正常返回结果，!recovered 区分两种错误
		if !normalReturn && !recovered {
			c.err = errGoexit
		}

        // 后续就是简单的错误处理或直接返回（遍历有没有chan需要发送结果）
        // 判断发生 panic时，解除 wg 的阻塞和移除当前 key（保证下一次执行）后
        // 再触发panic
        ...
	}()

	func() { // 需要释放状态，所以先在此处捕获 panic
		defer func() {
			if !normalReturn {
				if r := recover(); r != nil {
					c.err = newPanicError(r)
				}
			}
		}()

		c.val, c.err = fn()
		normalReturn = true
	}()

    // 如果没有正常返回，运行到这说明一定发生了panic
    // 因为 normalReturn = true 没有被执行
	if !normalReturn { 
		recovered = true
	}
}
```

# 收尾
&emsp;&emsp;通过 newPanicError 捕获发生 panic 时的堆栈信息；

> tips:
```go
// 使输出的崩溃转储更详细
go panic(e)
select {} 
```