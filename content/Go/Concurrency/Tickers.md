---
title: "Go 并发：周期定时器 Ticker"
description: "从 Go by Example 的 Tickers 示例出发，理解 time.NewTicker、周期通道、Stop、select 退出，以及 time.Tick 的泄漏风险。"
tags:
  - Go
  - 编程语言
  - 并发
  - 定时器
created: 2026-10-29
modified: 2026-10-29
---

# Go 并发：周期定时器 Ticker

Timer 只触发一次，Ticker 则按固定间隔重复触发。它适合轮询、心跳、定时刷新和周期性任务。

[Go by Example 的 Tickers](https://gobyexample.com/tickers) 用 500ms 的 ticker 演示了周期触发和停止。本文补充 `time.Tick` 的泄漏风险、丢 tick 行为和退出模式。

## 完整示例

```go
package main

import (
	"fmt"
	"time"
)

func main() {
	ticker := time.NewTicker(500 * time.Millisecond)
	done := make(chan bool)

	go func() {
		for {
			select {
			case <-done:
				return
			case t := <-ticker.C:
				fmt.Println("Tick at", t)
			}
		}
	}()

	time.Sleep(1600 * time.Millisecond)
	ticker.Stop()
	done <- true
	fmt.Println("Ticker stopped")
}
```

运行（时间戳取决于运行时刻）：

```bash
$ go run tickers.go
Tick at 2026-10-29 15:00:00.487625 +0800 CST
Tick at 2026-10-29 15:00:00.988063 +0800 CST
Tick at 2026-10-29 15:00:01.488076 +0800 CST
Ticker stopped
```

大约 1600ms 内触发 3 次。

## 创建 Ticker

```go
ticker := time.NewTicker(500 * time.Millisecond)
defer ticker.Stop()
```

`ticker.C` 是 `<-chan time.Time`，每隔指定间隔收到一个时间值。

Ticker 会持续触发，直到调用 `Stop`。忘记停止会导致底层资源无法释放，尤其是长期运行的服务。

## 用 `select` 退出

Ticker 通常和 `select` 一起使用，以便同时监听退出信号：

```go
done := make(chan bool)

go func() {
	for {
		select {
		case <-done:
			return
		case t := <-ticker.C:
			fmt.Println("tick:", t)
		}
	}
}()

// 稍后
ticker.Stop()
done <- true
```

这里有两个退出路径：

- `ticker.Stop()` 停止 ticker；
- `done` channel 通知 goroutine 返回。

只调用 `Stop` 而不通知 goroutine，`select` 可能永久阻塞在 `ticker.C` 上（因为停止后不会再发送值）。所以需要一个独立的退出信号。

## `time.Tick` 的便利与风险

`time.Tick(d)` 是创建 ticker 并返回 channel 的快捷方式：

```go
for t := range time.Tick(time.Second) {
	fmt.Println(t)
}
```

它很方便，但**无法停止**，底层 ticker 也不会被垃圾回收。对于长期运行的程序，这是资源泄漏。

只有在程序生命周期很短、或确定要永久运行时才使用 `time.Tick`。生产代码更推荐：

```go
ticker := time.NewTicker(time.Second)
defer ticker.Stop()
```

## tick 会丢失

Ticker 的 channel 容量为 1。如果接收方处理太慢，两次 tick 之间没有及时接收，中间的 tick 会被丢弃。

这意味着：

- Ticker 不保证每个间隔都触发一次；
- 它保证“尽可能按间隔提供最新时间”；
- 不能把 tick 当作可靠的消息队列。

如果需要严格处理每个事件，应该用队列缓冲，或根据时间戳补偿。

## 常见错误

### 1. 忘记 `Stop`

长期运行的 ticker 不停止会持续占用资源。使用 `defer ticker.Stop()` 或显式停止。

### 2. 用 `time.Tick` 创建长期任务

`time.Tick` 无法停止，可能造成泄漏。使用 `time.NewTicker`。

### 3. 停止 ticker 后仍等待 `ticker.C`

停止后 channel 不会关闭，也不会再发送值。需要用独立的 done channel 退出 goroutine。

### 4. 假设每个 tick 都会到达

接收慢时 tick 会丢失。不要把它当作可靠队列。

### 5. 在 ticker goroutine 中执行长时间阻塞操作

如果处理时间超过间隔，tick 会堆积/丢弃，周期也会漂移。耗时任务应交给 worker 或异步处理。

### 6. 关闭 ticker 的 channel

不要关闭 `ticker.C`。它由运行时管理，停止 ticker 应使用 `Stop`，而不是关闭 channel。

## 和 Java 对照

| 概念 | Go | Java |
| --- | --- | --- |
| 周期任务 | `time.NewTicker` | `scheduleAtFixedRate` |
| 停止 | `Ticker.Stop` | `ScheduledFuture.cancel` |
| 固定延迟 | 手动控制 | `scheduleWithFixedDelay` |
| 丢 tick | 接收慢时可能丢失 | 取决于线程池策略 |
| 快捷方式 | `time.Tick` | 无直接等价 |

Java 的 `scheduleAtFixedRate` 会尽量按固定频率执行，任务超时会影响后续调度；Go 的 ticker 只负责发信号，具体执行由接收方决定。

## 动手练习

1. 创建 500ms ticker，打印 3 次后停止。
2. 用 `select` 同时监听 ticker 和 done channel。
3. 用 `defer ticker.Stop()` 改写停止逻辑。
4. 在 ticker 回调中睡眠 1 秒，观察 tick 丢失。
5. 用 `time.Tick` 写一个简单循环，再改成可停止的 `NewTicker`。
6. 用 ticker 实现一个每 2 秒刷新一次的缓存任务。

## 小结

- `time.NewTicker(d)` 按间隔重复发送时间值。
- `Stop` 停止 ticker，但不会关闭 channel。
- Ticker 通常和 `select`、done channel 一起使用。
- `time.Tick` 无法停止，长期使用可能泄漏。
- 接收慢时 tick 会丢失，Ticker 不是可靠队列。
- 耗时任务不要直接在 ticker 循环中阻塞执行。

---

上一篇：[[Go/Concurrency/Timers|Go 并发：定时器 Timer]]
下一篇：[[Go/Concurrency/Worker Pools|Go 并发：工作池]]

返回 [[Go/Concurrency/index|Go 并发]]。
