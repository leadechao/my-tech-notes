---
title: "Go 并发：range 遍历 channel"
description: "从 Go by Example 的 Range over Channels 示例出发，理解关闭 channel 后 range 如何读取剩余值并结束，以及常见的关闭规则。"
tags:
  - Go
  - 编程语言
  - 并发
  - channel
created: 2026-10-29
modified: 2026-10-29
---

# Go 并发：range 遍历 channel

`for range` 不仅可以遍历数组、切片、map 和字符串，也可以遍历 channel。它会持续接收值，直到 channel 被关闭。

这是消费 channel 最常用的写法，尤其是在 worker pool、生产者-消费者和事件流中。

[Go by Example 的 Range over Channels](https://gobyexample.com/range-over-channels) 用一个带缓冲的队列演示了关闭后读取剩余值。本文补充关闭规则、`ok` 判断和常见死锁。

## 完整示例

```go
package main

import "fmt"

func main() {
	queue := make(chan string, 2)
	queue <- "one"
	queue <- "two"
	close(queue)

	for elem := range queue {
		fmt.Println(elem)
	}
}
```

运行：

```bash
$ go run range-over-channels.go
one
two
```

## `range` 的语义

```go
for elem := range queue {
	fmt.Println(elem)
}
```

它等价于反复执行：

```go
for {
	elem, ok := <-queue
	if !ok {
		break
	}
	fmt.Println(elem)
}
```

区别是 `range` 语法更简洁。

`range` 会在以下情况结束：

- channel 被关闭，且缓冲区中的值已经全部接收；
- 循环体 `break` 或函数 `return`。

如果 channel 没有关闭，`range` 会一直阻塞等待新值。

## 关闭非空 channel

示例中先发送两个值，再 `close(queue)`，然后才 `range`。这仍然能读到 `one` 和 `two`。

关闭 channel 表示“不会再有新值”，但**已缓冲的值仍然可以被接收**。接收完剩余值后，`range` 才会结束。

这一点和“关闭即丢弃”不同。关闭是生产者发出的完成信号，不是清空操作。

## 直接接收与 `ok`

不使用 `range` 时，可以用两个返回值判断 channel 是否已关闭：

```go
v, ok := <-queue
if !ok {
	fmt.Println("channel closed")
}
```

- `ok == true`：成功接收到一个值；
- `ok == false`：channel 已关闭且没有剩余值，此时 `v` 是元素类型的零值。

只接收一个值时，无法区分“收到零值”和“channel 已关闭”：

```go
v := <-queue // 无法判断是否关闭
```

需要区分时使用 `v, ok := <-queue`。

## 谁来关闭 channel

规则很简单：**发送方负责关闭，接收方不要关闭**。

```go
func producer(ch chan<- int) {
	defer close(ch)
	for i := 0; i < 3; i++ {
		ch <- i
	}
}
```

接收方只负责读取：

```go
for v := range ch {
	fmt.Println(v)
}
```

如果接收方关闭 channel，而发送方还在发送，就会触发 panic：

```text
panic: send on closed channel
```

如果有多个发送方，通常需要额外的协调（例如 WaitGroup）确保所有发送方都完成后，再由一个协调者关闭。

## nil channel 与已关闭 channel

- **nil channel**：`range` 会永久阻塞；
- **已关闭且为空**：`range` 立即结束；
- **已关闭但还有缓冲值**：先读完剩余值，再结束。

```go
var ch chan int
for range ch {
	// 永远不会执行
}
```

nil channel 的 `range` 不会 panic，只会阻塞。实际代码中如果不小心把 channel 留成 nil，很容易造成死锁。

## 常见错误

### 1. 忘记关闭 channel

```go
ch := make(chan int)
go func() {
	ch <- 1
	ch <- 2
	// 忘记 close(ch)
}()

for v := range ch {
	fmt.Println(v)
}
```

发送完两个值后，`range` 会继续等待第三个值，程序永远不结束。发送方完成工作后应关闭 channel。

### 2. 接收方关闭 channel

接收方不知道是否还有其他发送方。关闭后如果发送方继续发送，会 panic。

### 3. 向已关闭的 channel 发送

会 panic。关闭前要确保没有发送方还会写入。

### 4. 重复关闭

关闭已关闭的 channel 会 panic。通常只由唯一的所有者关闭。

### 5. 以为关闭会丢弃缓冲值

关闭后仍能读取缓冲区中的值。只有读完之后，`range` 才结束。

### 6. 用 `v := <-ch` 判断关闭

单返回值无法区分零值和关闭。使用 `v, ok := <-ch` 或 `range`。

## 和 Java 对照

| 概念 | Go | Java |
| --- | --- | --- |
| 遍历队列 | `for v := range ch` | `while ((v = queue.take()) != null)` |
| 结束信号 | `close(ch)` | poison pill / 中断 |
| 剩余元素 | 关闭后仍可读取 | 取决于队列实现 |
| 判断结束 | `ok == false` | `null` / 特殊值 |
| 多生产者 | 协调后关闭 | 协调后发送结束信号 |

Java 的 `BlockingQueue` 没有“关闭”概念，通常用 poison pill（特殊值）表示结束。Go 的 `close` 是语言级信号，更直接，但也要求严格遵守“发送方关闭”的规则。

## 动手练习

1. 创建带缓冲 channel，发送 3 个值后关闭，用 `range` 读取。
2. 用 `v, ok := <-ch` 手动实现 `range` 的等价逻辑。
3. 写一个生产者函数，用 `defer close(ch)` 关闭 channel。
4. 启动两个生产者写入同一个 channel，用 WaitGroup 等待后由一个协调者关闭。
5. 构造一个忘记关闭 channel 的程序，观察 `range` 阻塞。
6. 对 nil channel 使用 `range`，观察程序行为。

## 小结

- `for range ch` 持续接收，直到 channel 关闭且缓冲值读完。
- 关闭 channel 不会丢弃已缓冲的值。
- 只有发送方应该关闭 channel；接收方不应关闭。
- `v, ok := <-ch` 可以判断 channel 是否已关闭。
- nil channel 的 `range` 会永久阻塞。
- 忘记关闭是 `range` 卡住的常见原因。

---

上一篇：[[Go/Concurrency/Channels|Go 并发：channel]]
下一篇：[[Go/Concurrency/Timers|Go 并发：定时器 Timer]]

返回 [[Go/Concurrency/index|Go 并发]]。
