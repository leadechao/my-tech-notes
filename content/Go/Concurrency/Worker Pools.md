---
title: "Go 并发：工作池"
description: "从 Go by Example 的 Worker Pools 示例出发，用 jobs/results channel 构建固定数量的 worker，理解关闭、收集与并发控制。"
tags:
  - Go
  - 编程语言
  - 并发
  - 工作池
created: 2026-10-29
modified: 2026-10-29
---

# Go 并发：工作池

工作池（worker pool）用固定数量的 goroutine 消费任务队列。它既能利用多核并发，又能限制同时运行的任务数，避免无限制创建 goroutine 压垮系统。

Go 的实现非常直接：一个 jobs channel 分发任务，一个 results channel 收集结果，若干个 worker 循环消费。

[Go by Example 的 Worker Pools](https://gobyexample.com/worker-pools) 用 3 个 worker 处理 5 个任务。本文补充关闭语义、结果收集、错误处理和常见死锁。

## 完整示例

```go
package main

import (
	"fmt"
	"time"
)

func worker(id int, jobs <-chan int, results chan<- int) {
	for j := range jobs {
		fmt.Println("worker", id, "started  job", j)
		time.Sleep(time.Second)
		fmt.Println("worker", id, "finished job", j)
		results <- j * 2
	}
}

func main() {
	const numJobs = 5
	jobs := make(chan int, numJobs)
	results := make(chan int, numJobs)

	for w := 1; w <= 3; w++ {
		go worker(w, jobs, results)
	}

	for j := 1; j <= numJobs; j++ {
		jobs <- j
	}
	close(jobs)

	for a := 1; a <= numJobs; a++ {
		<-results
	}
}
```

运行（worker 的先后顺序不固定）：

```bash
$ time go run worker-pools.go
worker 1 started  job 1
worker 2 started  job 2
worker 3 started  job 3
worker 1 finished job 1
worker 1 started  job 4
worker 2 finished job 2
worker 2 started  job 5
worker 3 finished job 3
worker 1 finished job 4
worker 2 finished job 5

real    0m2.358s
```

5 个任务各耗时 1 秒，3 个 worker 并发执行，总耗时约 2 秒。

## 结构

```go
func worker(id int, jobs <-chan int, results chan<- int) {
	for j := range jobs {
		// 处理任务
		results <- j * 2
	}
}
```

- `jobs <-chan int`：只接收任务；
- `results chan<- int`：只发送结果；
- `for j := range jobs`：持续消费，直到 jobs 被关闭。

方向类型能在编译期防止 worker 误写 jobs 或误读 results。

## 启动 worker

```go
for w := 1; w <= 3; w++ {
	go worker(w, jobs, results)
}
```

这里启动 3 个 worker。它们一开始都阻塞在 `range jobs` 上，等待任务。

worker 数量通常根据 CPU 核数、I/O 特性和下游承载能力决定。CPU 密集型任务可以接近 `runtime.NumCPU()`；I/O 密集型可以更多，但要避免压垮外部服务。

## 发送任务并关闭

```go
for j := 1; j <= numJobs; j++ {
	jobs <- j
}
close(jobs)
```

`close(jobs)` 表示“不会再有新任务”。worker 的 `range` 在读完剩余任务后退出。

如果不关闭 jobs，worker 会永远阻塞在 `range` 上，程序无法正常结束。

## 收集结果

```go
for a := 1; a <= numJobs; a++ {
	<-results
}
```

这里按任务数量接收结果。收到 `numJobs` 个结果后，主 goroutine 继续或退出。

这种写法要求“每个任务恰好产生一个结果”。如果某个任务失败、跳过或产生多个结果，接收次数就会不匹配，导致阻塞或数据丢失。

更健壮的写法是用 WaitGroup 等待所有 worker 结束，然后关闭 results：

```go
var wg sync.WaitGroup

for w := 1; w <= 3; w++ {
	wg.Add(1)
	go func(id int) {
		defer wg.Done()
		worker(id, jobs, results)
	}(w)
}

go func() {
	wg.Wait()
	close(results)
}()

for r := range results {
	fmt.Println(r)
}
```

这样就不需要预先知道结果数量，results 关闭后 `range` 自动结束。

## 结果顺序

worker 并发执行，结果到达顺序通常与任务编号无关。如果业务要求有序结果，可以：

- 在结果中带上任务 ID，收集后排序；
- 用固定大小的结果槽位按索引写入；
- 改用有序的流水线设计。

不要假设 `results` 的顺序与 `jobs` 一致。

## 错误处理

示例只传递了结果，没有传递错误。生产代码通常需要：

```go
type result struct {
	jobID int
	value int
	err   error
}
```

worker 把错误和结果一起发送，主 goroutine 统一处理。如果需要“任一任务失败就取消其他任务”，使用 `context.Context` 和 `errgroup`。

## 常见错误

### 1. 忘记关闭 jobs

worker 永远阻塞在 `range jobs`，主 goroutine 可能也在等待结果，最终死锁。

### 2. 结果数量不匹配

发送 N 个任务却只接收 N-1 个结果，或者某个任务没有发送结果，都会导致接收方阻塞。

### 3. results 未缓冲且收集太晚

如果 results 无缓冲，worker 发送结果时会阻塞。主 goroutine 必须及时接收，或者给 results 足够缓冲。

### 4. worker 泄漏

jobs 没有关闭、下游阻塞、panic 未处理，都会导致 worker goroutine 永久停留。

### 5. worker 数量过多

无限制增加 worker 不会无限提升吞吐量，反而会增加调度和内存开销，甚至压垮下游。

### 6. 共享状态没有同步

多个 worker 同时写同一个 map 或切片，会产生数据竞争。让每个 worker 返回结果，由主 goroutine 汇总，或者加锁。

## 和 Java 对照

| 概念 | Go | Java |
| --- | --- | --- |
| 任务队列 | jobs channel | `BlockingQueue` |
| 工作线程 | 固定数量 goroutine | `ThreadPoolExecutor` |
| 提交任务 | 发送到 jobs | `execute` / `submit` |
| 收集结果 | results channel | `Future.get` / `CompletionService` |
| 关闭 | `close(jobs)` | `shutdown()` |
| 取消 | context | `Future.cancel` |

Java 的线程池把任务提交、线程管理和结果获取封装在 API 中；Go 需要自己用 channel 搭建，但结构清晰、控制力更强。

## 动手练习

1. 实现 3 个 worker 处理 5 个任务，每个任务耗时 1 秒。
2. 给结果加上任务 ID，按 ID 排序后输出。
3. 用 WaitGroup + close(results) 改写结果收集。
4. 让 worker 返回错误，统计失败任务数。
5. 用 context 在某个任务失败时取消其他任务。
6. 把 worker 数量改成 1 和 10，观察总耗时和资源使用。

## 小结

- 工作池用固定数量的 worker 消费 jobs channel。
- worker 用 `range jobs` 消费，`close(jobs)` 表示任务结束。
- results channel 收集结果，数量必须与任务匹配。
- WaitGroup + close(results) 是更健壮的结束方式。
- 结果顺序不固定，需要有序时显式处理。
- 限制 worker 数量，避免无限制并发。

---

上一篇：[[Go/Concurrency/Tickers|Go 并发：周期定时器 Ticker]]
下一篇：[[Go/Concurrency/WaitGroups|Go 并发：WaitGroup]]

返回 [[Go/Concurrency/index|Go 并发]]。
