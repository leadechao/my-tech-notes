---
title: "Go 并发：WaitGroup"
description: "从 Go by Example 的 WaitGroups 示例出发，用 Add/Done/Wait 等待一组 goroutine，并了解较新的 wg.Go 写法、复制禁忌与错误传播。"
tags:
  - Go
  - 编程语言
  - 并发
  - 同步
created: 2026-10-29
modified: 2026-10-29
---

# Go 并发：WaitGroup

启动多个 goroutine 后，主 goroutine 需要一种方式等待它们全部完成。`sync.WaitGroup` 就是为此设计的计数器。

它的用法可以概括为：

- `Add(n)`：增加待完成任务数；
- `Done()`：完成一个任务，计数减一；
- `Wait()`：阻塞到计数归零。

[Go by Example 的 WaitGroups](https://gobyexample.com/waitgroups) 用 5 个 worker 演示了等待。本文以传统的 `Add/Done` 写法为主，并补充较新的 `wg.Go`、复制禁忌和错误传播。

## 传统写法

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

func worker(id int) {
	fmt.Printf("Worker %d starting\n", id)
	time.Sleep(time.Second)
	fmt.Printf("Worker %d done\n", id)
}

func main() {
	var wg sync.WaitGroup

	for i := 1; i <= 5; i++ {
		wg.Add(1)
		go func(id int) {
			defer wg.Done()
			worker(id)
		}(i)
	}

	wg.Wait()
}
```

运行（顺序不固定）：

```bash
$ go run waitgroups.go
Worker 5 starting
Worker 3 starting
Worker 4 starting
Worker 1 starting
Worker 2 starting
Worker 4 done
Worker 1 done
Worker 2 done
Worker 5 done
Worker 3 done
```

## 为什么需要 WaitGroup

如果只在 `main` 里启动 goroutine：

```go
for i := 1; i <= 5; i++ {
	go worker(i)
}
```

`main` 可能立刻返回，导致所有 worker 被终止。`wg.Wait()` 会阻塞到所有任务完成。

也可以用一个 `done` channel 等待，但 WaitGroup 更适合“一组数量已知的任务”。

## Add、Done、Wait

```go
var wg sync.WaitGroup

wg.Add(1)
go func() {
	defer wg.Done()
	// 工作
}()

wg.Wait()
```

关键点：

- **`Add` 必须在启动 goroutine 之前调用**；
- `Done` 通常用 `defer`，确保 panic 或提前返回时也能减计数；
- `Wait` 可以多次调用，但不要在 `Add` 并发进行时调用。

如果 `Add` 放在 goroutine 内部：

```go
go func() {
	wg.Add(1) // 错误：可能 Add 还没执行，Wait 已经返回
	defer wg.Done()
}()
```

`Wait` 可能在 `Add` 之前就执行，导致提前返回。

## 较新的 `wg.Go`

较新的 Go 版本提供了 `WaitGroup.Go`，把 `Add(1)`、启动 goroutine 和 `Done` 封装在一起：

```go
var wg sync.WaitGroup

for i := 1; i <= 5; i++ {
	wg.Go(func() {
		worker(i)
	})
}

wg.Wait()
```

`wg.Go` 会先增加计数，再启动 goroutine，并在函数返回时自动 `Done`。它减少了忘记 `Add` 或 `Done` 的机会。

如果项目需要兼容旧版本 Go，继续使用传统的 `Add` + `go` + `defer Done`。Go by Example 的示例使用了较新的 `wg.Go` 写法。

## 循环变量

Go 1.22 起，循环变量每次迭代都是新变量，因此：

```go
for i := 1; i <= 5; i++ {
	wg.Go(func() {
		worker(i) // 每个 goroutine 捕获自己的 i
	})
}
```

在 Go 1.21 及更早版本中，所有 goroutine 会共享同一个 `i`，通常需要显式传参：

```go
wg.Add(1)
go func(id int) {
	defer wg.Done()
	worker(id)
}(i)
```

## WaitGroup 不能复制

`sync.WaitGroup` 内部包含计数器状态，复制后两个副本会各自计数，导致 `Wait` 永远不返回或 panic。

```go
func run(wg sync.WaitGroup) { // 错误：复制
	wg.Wait()
}
```

应该传指针：

```go
func run(wg *sync.WaitGroup) {
	wg.Wait()
}
```

同样，包含 WaitGroup 的结构体也不要按值传递。

## 计数不能变成负数

如果 `Done` 调用次数多于 `Add`，会 panic：

```text
panic: sync: negative WaitGroup counter
```

确保每个 `Add(1)` 恰好对应一个 `Done()`。

## 错误传播

WaitGroup 只负责等待，不负责收集错误。如果 worker 可能失败，需要额外的错误通道：

```go
errCh := make(chan error, 1)

wg.Go(func() {
	if err := doWork(); err != nil {
		select {
		case errCh <- err:
		default:
		}
	}
})

wg.Wait()
close(errCh)

for err := range errCh {
	fmt.Println("error:", err)
}
```

更完整的方案是 `golang.org/x/sync/errgroup`，它把 WaitGroup、错误传播和 context 取消组合在一起。

## 常见错误

### 1. 忘记 `Wait`

主 goroutine 提前退出，worker 被终止。

### 2. `Add` 放在 goroutine 内

存在竞态，`Wait` 可能在 `Add` 前返回。

### 3. 忘记 `Done`

计数永远不归零，`Wait` 永久阻塞。使用 `defer wg.Done()`。

### 4. 复制 WaitGroup

计数状态被复制，行为不可预测。始终传指针。

### 5. `Done` 次数过多

计数变成负数，直接 panic。

### 6. 用 WaitGroup 收集错误

WaitGroup 不保存错误。需要错误通道或 errgroup。

## 和 Java 对照

| 概念 | Go | Java |
| --- | --- | --- |
| 等待一组任务 | `sync.WaitGroup` | `CountDownLatch` / `Phaser` |
| 计数 | `Add` / `Done` | `countDown` / `await` |
| 等待全部完成 | `Wait` | `await` / `CompletableFuture.allOf` |
| 错误传播 | 手动 / errgroup | `Future.get` / `CompletionException` |
| 复制 | 禁止 | 对象引用 |
| 取消 | context | `Future.cancel` / 中断 |

Java 的 `CountDownLatch` 是一次性的，`Phaser` 更灵活；Go 的 WaitGroup 可以复用，但复用时要特别小心并发 `Add`。

## 动手练习

1. 启动 5 个 worker，用 WaitGroup 等待全部完成。
2. 用 `wg.Go` 改写传统写法。
3. 去掉 `defer wg.Done()`，观察 `Wait` 阻塞。
4. 把 `wg` 按值传给函数，观察复制问题。
5. 用错误通道收集 worker 错误。
6. 用 errgroup 改写，支持任一任务失败时取消其他任务。

## 小结

- WaitGroup 用 `Add`、`Done`、`Wait` 等待一组 goroutine。
- `Add` 必须在启动 goroutine 前调用，`Done` 用 defer 保证执行。
- 新版本可以用 `wg.Go` 简化 Add/Go/Done。
- WaitGroup 不能复制，传参用指针。
- 计数不能为负，`Done` 次数不能多于 `Add`。
- WaitGroup 不传播错误，需要错误通道或 errgroup。

---

上一篇：[[Go/Concurrency/Worker Pools|Go 并发：工作池]]
下一篇：[[Go/Concurrency/Rate Limiting|Go 并发：限流]]

返回 [[Go/Concurrency/index|Go 并发]]。
