---
title: "Go 并发：channel"
description: "从 Go by Example 的 Channels 示例出发，理解 channel 的发送、接收、阻塞、缓冲、关闭、方向与死锁。"
tags:
  - Go
  - 编程语言
  - 并发
  - channel
created: 2026-10-29
modified: 2026-10-29
---

# Go 并发：channel

channel 是连接 goroutine 的管道。一个 goroutine 发送值，另一个 goroutine 接收值。它既是通信机制，也是同步机制。

Go 的并发哲学是：“不要通过共享内存来通信，而要通过通信来共享内存。” channel 就是这句话的核心工具。

[Go by Example 的 Channels](https://gobyexample.com/channels) 用 ping 消息演示了最基本的发送和接收。本文补充缓冲、关闭、方向、nil channel 和死锁。

## 完整示例

```go
package main

import "fmt"

func main() {
	messages := make(chan string)

	go func() { messages <- "ping" }()

	msg := <-messages
	fmt.Println(msg)
}
```

运行：

```bash
$ go run channels.go
ping
```

## 创建 channel

```go
messages := make(chan string)
```

`chan string` 表示“传递 string 的 channel”。channel 是强类型的，不能把 int 发进 `chan string`。

channel 的零值是 nil。nil channel 的发送和接收都会永久阻塞（后面会讲）。

## 发送与接收

```go
messages <- "ping" // 发送
msg := <-messages  // 接收
```

箭头方向表示数据流向：

- `ch <- v`：把 `v` 发送到 `ch`；
- `v := <-ch`：从 `ch` 接收一个值；
- `<-ch`：接收并丢弃值，常用于等待信号。

## 默认是无缓冲的

```go
messages := make(chan string)
```

这是无缓冲 channel。发送和接收必须“配对”：

- 发送方会阻塞，直到有接收方准备好；
- 接收方会阻塞，直到有发送方准备好。

示例中：

```go
go func() { messages <- "ping" }()
msg := <-messages
```

goroutine 发送时可能先阻塞，直到 `main` 执行 `<-messages`；两者配对后，值传递完成，双方继续。

这种阻塞特性让 channel 天然具备同步能力。不需要额外的锁或睡眠，就能等待另一个 goroutine 完成。

## 用 channel 等待完成

```go
done := make(chan bool)

go func() {
	// 做一些工作
	done <- true
}()

<-done
```

这比 `time.Sleep` 可靠：主 goroutine 会一直等到工作完成。

如果不需要传值，可以用 `chan struct{}`，它不占额外空间：

```go
done := make(chan struct{})
close(done) // 关闭也可以作为广播信号
```

## 缓冲 channel

给 `make` 传第二个参数可以创建带缓冲的 channel：

```go
ch := make(chan string, 2)

ch <- "a"
ch <- "b"
fmt.Println(<-ch) // a
fmt.Println(<-ch) // b
```

缓冲区未满时，发送不会阻塞；缓冲区为空时，接收会阻塞。容量为 2 时，可以连续发送两个值而不需要接收方。

缓冲 channel 不是无限队列。容量满后，发送方仍会阻塞。

## 关闭 channel

发送方可以用 `close` 表示“不会再有值了”：

```go
ch := make(chan int, 2)
ch <- 1
ch <- 2
close(ch)

for v := range ch {
	fmt.Println(v)
}
// 1
// 2
```

关闭后：

- 已缓冲的值仍可以被接收；
- 接收完所有值后，再接收会立即返回零值，第二个返回值为 `false`；
- `range` 会在通道关闭且值取完后结束。

```go
v, ok := <-ch
if !ok {
	fmt.Println("channel closed")
}
```

规则：

- **只有发送方应该关闭 channel**；
- 不要关闭一个已经关闭的 channel（panic）；
- 不要向已关闭的 channel 发送值（panic）；
- 接收方通常不负责关闭。

## channel 方向

函数参数可以限制 channel 的方向：

```go
func worker(jobs <-chan int, results chan<- int) {
	for j := range jobs {
		results <- j * 2
	}
}
```

- `<-chan int`：只能接收；
- `chan<- int`：只能发送；
- `chan int`：双向。

方向限制能在编译期防止误用，建议在函数签名中尽量使用单向类型。

## nil channel

nil channel 的发送和接收都会永久阻塞：

```go
var ch chan int
ch <- 1  // 永久阻塞
<-ch     // 永久阻塞
```

这看起来没用，但在 `select` 中很有价值：把一个 channel 设为 nil，可以让对应的 case 永久不可选，从而动态禁用某个分支。

## 死锁

如果所有 goroutine 都在等待 channel，而没有任何一方能推进，运行时会报死锁：

```go
func main() {
	ch := make(chan int)
	ch <- 1 // 没有接收方
}
```

输出：

```text
fatal error: all goroutines are asleep - deadlock!
```

常见死锁原因：

- 无缓冲 channel 只发送不接收；
- 只接收不发送；
- 循环等待多个 channel；
- 忘记关闭 channel，导致 `range` 永远不结束。

## `select` 简介

需要同时等待多个 channel 时，使用 `select`：

```go
select {
case msg := <-ch1:
	fmt.Println("ch1:", msg)
case msg := <-ch2:
	fmt.Println("ch2:", msg)
case <-time.After(time.Second):
	fmt.Println("timeout")
}
```

`select` 会等待第一个就绪的 case；多个同时就绪时随机选择一个。超时、取消和多路复用都依赖它。

## 常见错误

### 1. 向已关闭的 channel 发送

会 panic。关闭前要确保没有发送方还会写入。

### 2. 重复关闭 channel

关闭已关闭的 channel 会 panic。通常只由唯一的发送方关闭。

### 3. 接收方关闭 channel

接收方不知道还有没有其他发送方，关闭可能导致发送方 panic。让发送方负责关闭。

### 4. 忘记关闭导致 `range` 卡住

`for v := range ch` 只有在 channel 关闭后才会结束。确保发送完成后调用 `close`。

### 5. 以为缓冲 channel 不会阻塞

缓冲区满后发送仍会阻塞。容量不是无限的。

### 6. 用 channel 传递大对象

channel 传递的是值副本。大结构体最好传指针，或重新设计数据流。

### 7. 在同一个 goroutine 中发送和接收无缓冲 channel

如果没有其他 goroutine 参与，会立即死锁。

## 和 Java 对照

| 概念 | Go | Java |
| --- | --- | --- |
| 通信机制 | channel | `BlockingQueue` |
| 无缓冲 | 默认 | `SynchronousQueue` |
| 缓冲 | `make(chan T, n)` | `ArrayBlockingQueue(n)` |
| 关闭 | `close(ch)` | 无直接等价，常用 poison pill |
| 多路等待 | `select` | `poll` / `take` + 多线程 |
| 方向 | `<-chan` / `chan<-` | 无编译期方向限制 |

Java 的 `BlockingQueue` 提供 `put` / `take`，语义接近缓冲 channel；Go 的 channel 还支持关闭广播和 `select` 多路复用，表达力更强。

## 动手练习

1. 用无缓冲 channel 在主 goroutine 和子 goroutine 之间传递一条消息。
2. 用 channel 等待子 goroutine 完成，替代 `time.Sleep`。
3. 创建容量为 3 的缓冲 channel，连续发送 3 个值，再接收。
4. 关闭一个带缓冲的 channel，用 `range` 读取剩余值。
5. 写一个只接收和只发送的函数参数，体会方向限制。
6. 构造一个死锁程序，观察运行时错误。
7. 用 `select` 同时等待两个 channel。

## 小结

- channel 是 goroutine 之间的类型化管道。
- 无缓冲 channel 的发送和接收必须配对，天然具备同步能力。
- 缓冲 channel 在容量范围内不阻塞，满后仍会阻塞。
- 发送方负责关闭；关闭后仍可读取剩余值。
- `<-chan` / `chan<-` 可以在编译期限制方向。
- nil channel 永久阻塞，可用于动态禁用 `select` 分支。
- 所有 goroutine 都阻塞时会触发死锁。

---

上一篇：[[Go/Concurrency/Goroutines|Go 并发：goroutine]]
下一篇：[[Go/Concurrency/Range over Channels|Go 并发：range 遍历 channel]]

返回 [[Go/Concurrency/index|Go 并发]]。
