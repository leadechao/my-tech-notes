---
title: "Go 并发：限流"
description: "从 Go by Example 的 Rate Limiting 示例出发，用 ticker 和带缓冲 channel 实现固定速率与突发限流，并了解生产级方案。"
tags:
  - Go
  - 编程语言
  - 并发
  - 限流
created: 2026-10-29
modified: 2026-10-29
---

# Go 并发：限流

限流控制单位时间内允许通过的操作数量。它可以保护下游服务、控制资源消耗、维持服务质量。

Go 用 goroutine、channel 和 ticker 就能实现基本的限流。核心思想是：用一个按固定间隔发送令牌的 channel，处理请求前先取一个令牌。

[Go by Example 的 Rate Limiting](https://gobyexample.com/rate-limiting) 演示了固定速率和允许突发两种模式。本文补充令牌桶、`time.Tick` 的泄漏风险和工程实践。

## 完整示例

```go
package main

import (
	"fmt"
	"time"
)

func main() {
	requests := make(chan int, 5)
	for i := 1; i <= 5; i++ {
		requests <- i
	}
	close(requests)

	limiter := time.Tick(200 * time.Millisecond)

	for req := range requests {
		<-limiter
		fmt.Println("request", req, time.Now())
	}

	burstyLimiter := make(chan time.Time, 3)
	for range 3 {
		burstyLimiter <- time.Now()
	}

	go func() {
		for t := range time.Tick(200 * time.Millisecond) {
			burstyLimiter <- t
		}
	}()

	burstyRequests := make(chan int, 5)
	for i := 1; i <= 5; i++ {
		burstyRequests <- i
	}
	close(burstyRequests)

	for req := range burstyRequests {
		<-burstyLimiter
		fmt.Println("request", req, time.Now())
	}
}
```

输出（时间戳省略）：

```text
request 1 ...
request 2 ...
request 3 ...
request 4 ...
request 5 ...
request 1 ...
request 2 ...
request 3 ...
request 4 ...
request 5 ...
```

第一批请求大约每 200ms 处理一个；第二批前三个立即处理，后两个各等待约 200ms。

## 固定速率限流

```go
limiter := time.Tick(200 * time.Millisecond)

for req := range requests {
	<-limiter
	handle(req)
}
```

`time.Tick(200ms)` 每 200ms 向 channel 发送一个时间值。处理请求前先 `<-limiter`，如果令牌还没到，就阻塞等待。

效果是：请求处理速度被限制为每 200ms 一个，也就是每秒 5 个。

这是一种“匀速”限流，不允许突发。

## 允许突发：令牌桶

现实中往往允许短时间的突发，只要长期平均速率不超过限制。可以用带缓冲的 channel 模拟令牌桶：

```go
burstyLimiter := make(chan time.Time, 3)
for range 3 {
	burstyLimiter <- time.Now()
}
```

初始放入 3 个令牌，表示允许最多 3 个突发请求。

后台 goroutine 每 200ms 补充一个令牌：

```go
go func() {
	for t := range time.Tick(200 * time.Millisecond) {
		burstyLimiter <- t
	}
}()
```

当缓冲区满时，补充操作会阻塞，因此最多只有 3 个令牌。

处理请求时取令牌：

```go
for req := range burstyRequests {
	<-burstyLimiter
	handle(req)
}
```

前 3 个请求立即通过，后续请求按 200ms 的速率通过。

## `time.Tick` 的资源问题

示例使用了 `time.Tick`，它简单但不能停止，底层 ticker 不会被回收：

```go
limiter := time.Tick(200 * time.Millisecond)
```

在长期运行的服务中，这会造成资源泄漏。生产代码推荐：

```go
ticker := time.NewTicker(200 * time.Millisecond)
defer ticker.Stop()

for req := range requests {
	<-ticker.C
	handle(req)
}
```

如果是后台补充令牌的 goroutine，还需要一个退出信号来停止 ticker。

## 生产级方案

手写 channel 限流适合学习和简单场景。生产环境通常使用：

- `golang.org/x/time/rate`：基于令牌桶的 `Limiter`，支持 `Allow`、`Wait`、`Reserve`；
- API 网关或中间件：按 IP、用户、接口限流；
- 分布式限流：Redis + Lua、令牌桶服务等。

`golang.org/x/time/rate` 的例子：

```go
limiter := rate.NewLimiter(5, 3) // 每秒 5 个，最多突发 3 个

if err := limiter.Wait(ctx); err != nil {
	return err
}
handle()
```

它比手写 channel 更灵活，也支持 context 取消。

## 限流与并发控制

限流和并发控制不是一回事：

- **限流**：控制单位时间的请求数量（QPS）；
- **并发控制**：控制同时运行的任务数量（信号量、worker pool）。

一个服务可能允许每秒 100 个请求，但同时只处理 10 个。需要根据场景组合使用。

## 常见错误

### 1. 用 `time.Tick` 长期运行

`time.Tick` 无法停止，可能泄漏。使用 `NewTicker` + `Stop`。

### 2. 以为限流等于限并发

限流控制速率，不控制同时运行数。高并发低 QPS 的场景仍可能耗尽资源。

### 3. 突发容量设置过大

缓冲区过大等于允许大量瞬时请求，可能压垮下游。根据下游承载能力设置。

### 4. 忽略 context 取消

等待令牌时如果请求已取消，应尽快返回。生产级 limiter 通常支持 context。

### 5. 在多个副本上各自限流

单机限流无法保证全局速率。分布式场景需要共享计数器或集中式限流。

### 6. 限流器 goroutine 泄漏

补充令牌的 goroutine 没有退出路径，长期运行会泄漏。提供 done channel 并停止 ticker。

## 和 Java 对照

| 概念 | Go | Java |
| --- | --- | --- |
| 令牌桶 | `x/time/rate` / channel | Guava `RateLimiter` |
| 固定速率 | ticker + 接收 | `RateLimiter.acquire` |
| 突发 | 带缓冲 channel | `RateLimiter` 的 permits |
| 并发控制 | worker pool / semaphore | `Semaphore` |
| 分布式限流 | Redis / 网关 | Redis / 网关 |

Java 的 Guava `RateLimiter` 直接提供令牌桶；Go 标准库没有内置限流器，通常使用 `x/time/rate` 或手写 channel。

## 动手练习

1. 用 ticker 实现每 200ms 处理一个请求。
2. 用带缓冲 channel 实现允许 3 个突发的限流器。
3. 把 `time.Tick` 改成 `time.NewTicker` 并正确停止。
4. 用 `x/time/rate` 实现每秒 5 个、突发 3 个的限流。
5. 同时限制 QPS 和并发数，观察两者差异。
6. 给限流器加上 context 取消。

## 小结

- 限流控制单位时间的请求数量，常用 ticker + channel 实现。
- 固定速率用无缓冲令牌 channel，突发用带缓冲令牌桶。
- `time.Tick` 无法停止，长期运行应使用 `NewTicker`。
- 生产环境优先使用 `x/time/rate` 或网关限流。
- 限流不等于并发控制，二者需要根据场景组合。
- 分布式限流需要共享状态，单机限流不够。

---

上一篇：[[Go/Concurrency/WaitGroups|Go 并发：WaitGroup]]
下一篇：[[Go/Concurrency/Atomic Counters|Go 并发：原子计数器]]

返回 [[Go/Concurrency/index|Go 并发]]。
