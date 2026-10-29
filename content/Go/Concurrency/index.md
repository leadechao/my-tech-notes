---
title: 并发
description: Go 并发学习笔记：goroutine、channel、定时器、工作池、WaitGroup、限流、原子操作与互斥锁。
created: 2026-10-29
modified: 2026-10-29
---

# Go 并发

Go 的并发模型建立在两个核心概念上：**goroutine** 负责并发执行，**channel** 负责通信与同步。

这一部分从 goroutine 和 channel 开始，逐步进入定时器、工作池、同步原语和并发模式。每篇都包含可运行示例、常见错误和 Java 对照。

## 文章

### 并发基础

- [[Go/Concurrency/Goroutines|Go 并发：goroutine]] —— `go` 关键字、调度、主 goroutine 退出、循环变量与数据竞争
- [[Go/Concurrency/Channels|Go 并发：channel]] —— 发送接收、阻塞、缓冲、关闭、方向、nil channel 与死锁
- [[Go/Concurrency/Range over Channels|Go 并发：range 遍历 channel]] —— 关闭语义、剩余值、`ok` 判断与发送方关闭规则

### 时间控制

- [[Go/Concurrency/Timers|Go 并发：定时器 Timer]] —— `NewTimer`、`Stop`、`time.After` 与超时控制
- [[Go/Concurrency/Tickers|Go 并发：周期定时器 Ticker]] —— 周期触发、`Stop`、丢 tick 与 `time.Tick` 泄漏

### 并发模式

- [[Go/Concurrency/Worker Pools|Go 并发：工作池]] —— jobs/results channel、固定 worker、关闭与结果收集
- [[Go/Concurrency/Rate Limiting|Go 并发：限流]] —— 固定速率、突发令牌桶、`x/time/rate` 与生产实践

### 同步与共享

- [[Go/Concurrency/WaitGroups|Go 并发：WaitGroup]] —— `Add`/`Done`/`Wait`、`wg.Go`、复制禁忌与错误传播
- [[Go/Concurrency/Atomic Counters|Go 并发：原子计数器]] —— `sync/atomic`、`Uint64`、CAS 与原子操作边界
- [[Go/Concurrency/Mutexes|Go 并发：互斥锁]] —— `sync.Mutex`、锁粒度、`RWMutex`、死锁与复制禁忌

## 学习建议

1. 先理解 goroutine 和 channel，再看定时器和并发模式。
2. 所有并发程序都用 `-race` 跑一遍，不要凭感觉判断安全。
3. 优先用 channel 表达任务流转，用 Mutex 保护确实需要共享的状态。
4. 不要无限制创建 goroutine，工作池和限流是常用手段。
5. 并发 bug 往往不可复现，写代码时就要把退出、关闭和错误路径想清楚。

---

上一阶段：[[Go/Advanced/index|Go 进阶]]

返回 [[Go/index|Go 学习笔记]]。
