---
title: "Go 并发：定时器 Timer"
description: "从 Go by Example 的 Timers 示例出发，理解 time.NewTimer、定时通道、Stop、time.After 与超时控制。"
tags:
  - Go
  - 编程语言
  - 并发
  - 定时器
created: 2026-10-29
modified: 2026-10-29
---

# Go 并发：定时器 Timer

Timer 表示未来某个时刻触发一次的定时器。它通过一个 channel 通知触发事件，可以和 `select` 组合实现超时、延迟和取消。

[Go by Example 的 Timers](https://gobyexample.com/timers) 演示了等待定时器触发和提前停止。本文补充 `time.After`、超时模式、`Reset` 注意事项和常见错误。

## 完整示例

```go
package main

import (
	"fmt"
	"time"
)

func main() {
	timer1 := time.NewTimer(2 * time.Second)

	<-timer1.C
	fmt.Println("Timer 1 fired")

	timer2 := time.NewTimer(time.Second)
	go func() {
		<-timer2.C
		fmt.Println("Timer 2 fired")
	}()

	stop2 := timer2.Stop()
	if stop2 {
		fmt.Println("Timer 2 stopped")
	}

	time.Sleep(2 * time.Second)
}
```

运行：

```bash
$ go run timers.go
Timer 1 fired
Timer 2 stopped
```

`timer1` 大约 2 秒后触发；`timer2` 在触发前被停止，所以不会打印 `Timer 2 fired`。

## 创建与等待

```go
timer := time.NewTimer(2 * time.Second)
<-timer.C
```

- `time.NewTimer(d)` 创建一个在 `d` 后触发的定时器；
- `timer.C` 是 `<-chan time.Time`；
- 从 `C` 接收会阻塞，直到定时器触发。

定时器只触发一次。需要周期性触发时，使用 `time.Ticker`。

## 停止定时器

```go
stop := timer.Stop()
```

`Stop` 返回布尔值：

- `true`：定时器在触发前被成功停止；
- `false`：定时器已经触发，或已经被停止。

停止后，定时器不会再向 `C` 发送值。`Stop` 不会关闭 channel。

注意竞态：如果定时器刚好在 `Stop` 前触发，`Stop` 可能返回 `false`，而另一个 goroutine 已经收到值。需要严格协调时，要结合 channel 接收和 `select` 处理。

## `time.Sleep` 与 Timer

如果只是阻塞当前 goroutine，`time.Sleep` 最简单：

```go
time.Sleep(2 * time.Second)
```

但 `time.Sleep` 无法取消，也不能和 `select` 组合。Timer 的优势是可以提前停止，或与其他事件一起等待：

```go
select {
case <-timer.C:
	fmt.Println("timeout")
case result := <-work:
	fmt.Println("done:", result)
}
```

## `time.After` 快捷方式

`time.After(d)` 等价于创建一个定时器并返回它的 channel：

```go
select {
case <-time.After(time.Second):
	fmt.Println("timeout")
case v := <-ch:
	fmt.Println(v)
}
```

适合一次性、短时间的超时。缺点是**无法停止**：即使超时分支没有走，底层定时器也要等到时间到才会释放。对长时间超时或高频调用，建议使用 `time.NewTimer` 并手动 `Stop`。

## `Reset` 的注意事项

Timer 可以重置：

```go
timer.Reset(2 * time.Second)
```

但在旧版本 Go 中，重置一个已经触发或停止的 timer 需要小心处理 channel 中的残留值。Go 1.23 对 timer channel 语义做了调整，具体行为应以当前版本文档为准。

如果不确定，优先创建新的 timer，而不是复用。这样代码更简单，也不容易踩到竞态。

## 常见错误

### 1. 以为 Timer 会周期性触发

Timer 只触发一次。周期任务使用 Ticker。

### 2. 忘记 `Stop`

不再需要定时器时，调用 `Stop` 释放资源。尤其是 `time.NewTimer` 创建的长定时器。

### 3. 用 `time.After` 创建大量长超时

`time.After` 无法停止，底层定时器会保留到触发。高频、长超时场景应使用 `NewTimer` + `Stop`。

### 4. 在多个 goroutine 中竞争接收 `C`

timer 的 channel 只发送一个值，多个接收者只有一个能拿到。需要广播时使用 `close` channel。

### 5. 假设 `Stop` 一定成功

如果定时器已经触发，`Stop` 返回 `false`。不要忽略返回值。

### 6. 用 `time.Sleep` 实现可取消的等待

`time.Sleep` 不能取消。需要取消时使用 Timer + `select` + context。

## 和 Java 对照

| 概念 | Go | Java |
| --- | --- | --- |
| 一次性延迟 | `time.NewTimer` | `ScheduledExecutorService.schedule` |
| 简单睡眠 | `time.Sleep` | `Thread.sleep` |
| 超时等待 | `select` + timer | `Future.get(timeout)` |
| 取消 | `Timer.Stop` | `Future.cancel` |
| 周期任务 | Ticker | `scheduleAtFixedRate` |

Java 的定时任务通常交给 `ScheduledExecutorService`，可以返回 `ScheduledFuture` 并取消。Go 的 Timer 更轻量，通常和 `select` 一起使用。

## 动手练习

1. 创建 2 秒 Timer，等待触发并打印。
2. 创建 1 秒 Timer，在触发前调用 `Stop`，验证不会打印。
3. 用 `select` 实现一个 500ms 超时。
4. 用 `time.After` 改写超时逻辑。
5. 创建 Timer，触发后调用 `Reset`，观察行为。
6. 比较 `time.Sleep` 和 Timer 在可取消性上的差异。

## 小结

- `time.NewTimer(d)` 创建一次性定时器，`C` 在触发时收到值。
- `Stop` 可以提前停止，返回是否成功。
- `time.After` 是快捷方式，但无法停止。
- Timer 常与 `select` 组合实现超时和取消。
- Timer 只触发一次，周期任务用 Ticker。
- 长超时、高频场景优先使用 `NewTimer` + `Stop`。

---

上一篇：[[Go/Concurrency/Range over Channels|Go 并发：range 遍历 channel]]
下一篇：[[Go/Concurrency/Tickers|Go 并发：周期定时器 Ticker]]

返回 [[Go/Concurrency/index|Go 并发]]。
