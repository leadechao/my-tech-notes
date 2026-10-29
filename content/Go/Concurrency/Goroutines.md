---
title: "Go 并发：goroutine"
description: "从 Go by Example 的 Goroutines 示例出发，理解 go 关键字、并发调度、主 goroutine 退出、等待方式与数据竞争。"
tags:
  - Go
  - 编程语言
  - 并发
  - goroutine
created: 2026-10-29
modified: 2026-10-29
---

# Go 并发：goroutine

goroutine 是 Go 的轻量级执行单元。在函数调用前加一个 `go` 关键字，就能让它在新的 goroutine 中并发执行。

和操作系统线程相比，goroutine 的创建成本很低，初始栈很小、按需增长，调度由 Go 运行时负责。因此一个程序可以轻松启动成千上万个 goroutine。

[Go by Example 的 Goroutines](https://gobyexample.com/goroutines) 用同步调用、goroutine 调用和匿名函数演示了基本用法。本文补充等待、退出、循环变量和竞态问题。

## 完整示例

```go
package main

import (
	"fmt"
	"time"
)

func f(from string) {
	for i := range 3 {
		fmt.Println(from, ":", i)
	}
}

func main() {
	f("direct")

	go f("goroutine")

	go func(msg string) {
		fmt.Println(msg)
	}("going")

	time.Sleep(time.Second)
	fmt.Println("done")
}
```

某次运行的输出：

```bash
$ go run goroutines.go
direct : 0
direct : 1
direct : 2
goroutine : 0
going
goroutine : 1
goroutine : 2
done
```

goroutine 的输出顺序不固定，可能和上面不同。

## `go` 关键字

普通调用是同步的：

```go
f("direct")
```

它会等 `f` 执行完才继续。前面加 `go` 就变成异步：

```go
go f("goroutine")
```

`go` 语句会立即返回，`f` 在另一个 goroutine 中执行。调用方继续往下走。

匿名函数也可以：

```go
go func(msg string) {
	fmt.Println(msg)
}("going")
```

注意末尾的 `("going")` 是立即传参，不是延迟调用。参数在启动 goroutine 时求值。

## 主 goroutine 退出会结束程序

`main` 函数本身运行在一个 goroutine 中。当 `main` 返回时，程序退出，所有其他 goroutine 都会被强制结束，不会等待它们完成。

```go
func main() {
	go fmt.Println("hello")
}
```

这个程序很可能什么都不输出，因为 `main` 在 goroutine 打印之前就返回了。

所以必须显式等待。示例里用了 `time.Sleep(time.Second)`，但这只是演示，不是可靠的同步方式。应该使用：

- `sync.WaitGroup`：等待一组 goroutine；
- channel：接收完成信号；
- `context`：取消和超时控制。

## goroutine 很轻量

goroutine 不是操作系统线程的一对一映射。Go 运行时会把大量 goroutine 调度到少量 OS 线程上，初始栈只有几 KB，按需增长。

这意味着：

- 启动 goroutine 比创建线程便宜；
- 不要因为“线程贵”而不敢并发；
- 但也不要无限制地创建，尤其是每个 goroutine 都持有大对象或执行阻塞 I/O 时。

需要限制并发度时，使用 worker pool、带缓冲的 channel 或 `golang.org/x/sync/semaphore`。

## 循环变量捕获

在 goroutine 中闭包捕获循环变量时，要特别注意版本差异。

Go 1.22 起，循环变量每次迭代都是新变量：

```go
for i := 0; i < 3; i++ {
	go func() {
		fmt.Println(i)
	}()
}
```

每个 goroutine 捕获各自的 `i`，输出 `0 1 2`（顺序不定）。

Go 1.21 及更早版本中，所有 goroutine 共享同一个 `i`，通常输出三个 `3`。旧代码需要显式传参：

```go
for i := 0; i < 3; i++ {
	go func(n int) {
		fmt.Println(n)
	}(i)
}
```

即使在新版本，显式传参也能让意图更清晰，尤其是在复杂循环中。

## 数据竞争

多个 goroutine 同时读写同一个变量，且没有同步，就会产生数据竞争：

```go
counter := 0
for i := 0; i < 1000; i++ {
	go func() {
		counter++ // 数据竞争
	}()
}
```

结果不可预测，可能小于 1000。使用 `go run -race` 或 `go test -race` 可以检测：

```bash
$ go run -race main.go
```

解决方式：

- 用 `sync.Mutex` 保护共享状态；
- 用 `sync/atomic` 做简单计数；
- 用 channel 传递数据，避免共享内存；
- 让每个 goroutine 操作自己的局部数据，最后合并。

## 常见错误

### 1. 忘记等待

`main` 返回后 goroutine 被终止。使用 WaitGroup、channel 或 `context` 等待。

### 2. 用 `time.Sleep` 做同步

睡眠时间无法保证 goroutine 完成，机器负载高时会失败。它只适合演示或非关键延迟。

### 3. 循环变量捕获错误

旧版本 Go 中所有 goroutine 共享循环变量。Go 1.22+ 已改善，但显式传参仍然更清晰。

### 4. 无限制创建 goroutine

每个请求启动一个 goroutine 且没有上限，可能耗尽内存或压垮下游。使用 worker pool 或信号量限流。

### 5. 共享变量没有同步

并发读写 map、slice、计数器等共享状态，必须加锁、使用原子操作或改为 channel 通信。

### 6. 在 goroutine 中 panic 未处理

一个 goroutine panic 会导致整个程序崩溃。关键 goroutine 应有 recover 或错误传递机制。

## 和 Java 对照

| 概念 | Go | Java |
| --- | --- | --- |
| 并发单元 | goroutine | Thread / 虚拟线程 |
| 启动方式 | `go f()` | `new Thread(...).start()` / Executor |
| 调度 | Go runtime 调度 | OS 线程 / 虚拟线程调度 |
| 初始栈 | 几 KB，按需增长 | 线程栈通常 MB 级 |
| 等待 | WaitGroup / channel | `join()` / `Future.get()` |
| 共享内存 | 需要显式同步 | 需要显式同步 |
| 通信 | channel | BlockingQueue 等 |

Java 21 的虚拟线程在“轻量”这一点上接近 goroutine，但 Go 把 goroutine 和 channel 作为语言内置机制，写法更统一。

## 动手练习

1. 同步调用 `f`，再用 `go f` 并发调用，观察输出顺序。
2. 用匿名 goroutine 打印一条消息，用 WaitGroup 等待它完成。
3. 启动 10 个 goroutine，各自打印编号，观察顺序。
4. 写一个共享计数器，不加锁运行 `-race`，观察数据竞争报告。
5. 用 Mutex 修复计数器，验证结果。
6. 用带缓冲 channel 限制同时运行的 goroutine 数量。

## 小结

- `go` 关键字启动 goroutine，调用立即返回。
- `main` 返回会终止所有 goroutine，必须显式等待。
- goroutine 轻量，适合大量并发，但需要限制并发度。
- Go 1.22 起循环变量每次迭代独立，旧版本需显式传参。
- 共享状态需要 Mutex、atomic 或 channel 同步。
- 使用 `-race` 检测数据竞争。

---

上一篇：[[Go/Advanced/Custom Errors|Go 进阶：自定义错误]]
下一篇：[[Go/Concurrency/Channels|Go 并发：channel]]

返回 [[Go/Concurrency/index|Go 并发]]。
