---
title: "Go 并发：原子计数器"
description: "从 Go by Example 的 Atomic Counters 示例出发，用 sync/atomic 的 Uint64、Add、Load 实现并发安全计数，并理解原子操作边界。"
tags:
  - Go
  - 编程语言
  - 并发
  - 原子操作
created: 2026-10-29
modified: 2026-10-29
---

# Go 并发：原子计数器

多个 goroutine 同时执行 `counter++` 会产生数据竞争，结果可能小于预期。对于简单的计数、标志位和指针替换，`sync/atomic` 提供了比锁更轻量的方案。

[Go by Example 的 Atomic Counters](https://gobyexample.com/atomic-counters) 用 50 个 goroutine 各递增 1000 次，验证原子操作能得到精确的 50000。本文补充原子类型、旧 API、复制禁忌和适用边界。

## 完整示例

```go
package main

import (
	"fmt"
	"sync"
	"sync/atomic"
)

func main() {
	var ops atomic.Uint64
	var wg sync.WaitGroup

	for range 50 {
		wg.Add(1)
		go func() {
			defer wg.Done()
			for range 1000 {
				ops.Add(1)
			}
		}()
	}

	wg.Wait()

	fmt.Println("ops:", ops.Load())
}
```

运行：

```bash
$ go run atomic-counters.go
ops: 50000
```

## 为什么需要原子操作

非原子自增看起来是一条语句，实际包含“读取、加一、写回”三个步骤。两个 goroutine 可能同时读到旧值，然后各自写回，导致一次更新丢失。

```go
var counter int
// 多个 goroutine 中：
counter++ // 数据竞争
```

结果通常小于 50000，而且每次运行可能不同。使用 `go run -race` 会报告数据竞争。

原子操作把“读取-修改-写入”作为一个不可分割的操作执行，保证不会互相覆盖。

## `atomic.Uint64`

```go
var ops atomic.Uint64
ops.Add(1)
fmt.Println(ops.Load())
```

常用方法：

- `Add(delta)`：原子加；
- `Load()`：原子读；
- `Store(v)`：原子写；
- `Swap(v)`：原子交换，返回旧值；
- `CompareAndSwap(old, new)`：比较并交换，返回是否成功。

`atomic.Uint64` 是 Go 1.19 引入的类型化原子值。旧代码常用函数式 API：

```go
var ops uint64
atomic.AddUint64(&ops, 1)
fmt.Println(atomic.LoadUint64(&ops))
```

两种方式语义相同。新代码优先使用类型化原子值，避免传错指针。

## 其他原子类型

标准库还提供：

- `atomic.Int32` / `atomic.Int64`；
- `atomic.Bool`；
- `atomic.Pointer[T]`；
- `atomic.Value`（可存储任意类型，但要求类型一致）。

例如，用 `atomic.Bool` 表示服务是否关闭：

```go
var closed atomic.Bool

if closed.CompareAndSwap(false, true) {
	fmt.Println("first closer wins")
}
```

`CompareAndSwap` 适合实现“只执行一次”的逻辑。

## 原子值不能复制

原子类型内部包含状态，复制后两个副本各自独立，失去同步意义：

```go
var a atomic.Uint64
b := a // 错误：复制原子值
```

包含原子值的结构体也不要按值传递。应使用指针，或把原子值作为包级变量。

`go vet` 通常会提示复制包含 `sync/atomic` 类型的结构体。

## 原子操作不是万能的

原子操作只适合单个变量的简单操作。以下场景应使用 Mutex 或 channel：

- 同时更新多个相关字段；
- 需要在多个步骤之间保持一致性；
- 需要保护 map、slice 等复合结构；
- 逻辑较复杂，原子操作难以表达。

例如，转账需要同时修改两个账户余额，单纯对每个余额做原子加减无法保证整体一致性，仍需要锁或事务。

## 常见错误

### 1. 用普通 `++` 代替原子操作

```go
ops++ // 数据竞争
```

必须使用 `ops.Add(1)` 或 `atomic.AddUint64`。

### 2. 复制原子值

复制后两个副本独立，计数不再共享。使用指针或包级变量。

### 3. 读取时不用 `Load`

直接读 `ops` 的字段在并发下不安全。使用 `Load()`。

### 4. 认为原子操作能保护复合状态

原子操作只保证单个操作不可分割，不保证多个字段之间的一致性。

### 5. 用原子操作替代所有锁

复杂临界区用 Mutex 更清晰。原子操作适合简单、无分支的更新。

### 6. 忽略 `-race`

原子操作正确不代表程序没有其他数据竞争。始终用 `-race` 检查。

## 和 Java 对照

| 概念 | Go | Java |
| --- | --- | --- |
| 原子整数 | `atomic.Int64` / `Uint64` | `AtomicLong` / `AtomicInteger` |
| 原子布尔 | `atomic.Bool` | `AtomicBoolean` |
| 原子引用 | `atomic.Pointer[T]` | `AtomicReference<T>` |
| 自增 | `Add(1)` | `incrementAndGet()` |
| CAS | `CompareAndSwap` | `compareAndSet` |
| 高并发计数 | 原子操作 / 分片 | `LongAdder` |

Java 的 `LongAdder` 在高竞争下比 `AtomicLong` 更快，Go 标准库没有直接等价物，通常用分片计数器或 `sync.Mutex` 实现。

## 动手练习

1. 用 `atomic.Uint64` 实现 50 个 goroutine 各加 1000 次。
2. 去掉原子操作，运行 `-race`，观察数据竞争报告。
3. 用旧式 `atomic.AddUint64` 改写。
4. 用 `atomic.Bool` 实现只执行一次的关闭逻辑。
5. 用 `atomic.Int64` 实现正负计数。
6. 比较原子操作和 Mutex 在简单计数场景下的性能。

## 小结

- 原子操作保证单个变量的读改写不可分割。
- `atomic.Uint64` 提供 `Add`、`Load`、`Store`、`Swap`、`CompareAndSwap`。
- 旧代码常用 `atomic.AddUint64` 等函数式 API。
- 原子值不能复制，包含原子值的结构体应传指针。
- 原子操作只适合简单状态，复合状态用 Mutex 或 channel。
- 始终用 `-race` 检查数据竞争。

---

上一篇：[[Go/Concurrency/Rate Limiting|Go 并发：限流]]
下一篇：[[Go/Concurrency/Mutexes|Go 并发：互斥锁]]

返回 [[Go/Concurrency/index|Go 并发]]。
