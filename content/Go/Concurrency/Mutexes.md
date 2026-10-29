---
title: "Go 并发：互斥锁"
description: "从 Go by Example 的 Mutexes 示例出发，用 sync.Mutex 保护共享 map，理解锁粒度、RWMutex、复制禁忌与死锁。"
tags:
  - Go
  - 编程语言
  - 并发
  - 互斥锁
created: 2026-10-29
modified: 2026-10-29
---

# Go 并发：互斥锁

多个 goroutine 同时读写共享数据时，需要同步。`sync.Mutex` 提供最基本的互斥锁：同一时刻只允许一个 goroutine 进入临界区。

对于计数器这类简单状态，原子操作更轻量；对于 map、结构体等复合状态，Mutex 更直观。

[Go by Example 的 Mutexes](https://gobyexample.com/mutexes) 用一个带锁的计数器容器演示了并发更新。本文补充锁粒度、RWMutex、复制禁忌和常见死锁。

## 完整示例

```go
package main

import (
	"fmt"
	"sync"
)

type Container struct {
	mu       sync.Mutex
	counters map[string]int
}

func (c *Container) inc(name string) {
	c.mu.Lock()
	defer c.mu.Unlock()
	c.counters[name]++
}

func main() {
	c := Container{
		counters: map[string]int{"a": 0, "b": 0},
	}

	var wg sync.WaitGroup
	wg.Add(3)

	doIncrement := func(name string, n int) {
		for range n {
			c.inc(name)
		}
	}

	go func() {
		defer wg.Done()
		doIncrement("a", 10000)
	}()

	go func() {
		defer wg.Done()
		doIncrement("a", 10000)
	}()

	go func() {
		defer wg.Done()
		doIncrement("b", 10000)
	}()

	wg.Wait()
	fmt.Println(c.counters)
}
```

运行：

```bash
$ go run mutexes.go
map[a:20000 b:10000]
```

两个 goroutine 各对 `a` 增加 10000 次，一个对 `b` 增加 10000 次，最终结果精确。

## 声明与使用

```go
type Container struct {
	mu       sync.Mutex
	counters map[string]int
}
```

`sync.Mutex` 的零值就是可用的未锁定状态，不需要初始化：

```go
var mu sync.Mutex
mu.Lock()
// 临界区
mu.Unlock()
```

在方法中通常用 `defer` 确保解锁：

```go
func (c *Container) inc(name string) {
	c.mu.Lock()
	defer c.mu.Unlock()
	c.counters[name]++
}
```

即使 `counters[name]++` panic，`defer` 也会执行解锁，避免其他 goroutine 永久阻塞。

## 保护共享状态

没有锁时：

```go
c.counters[name]++
```

多个 goroutine 同时执行，可能丢失更新，甚至触发 `fatal error: concurrent map writes`。

加锁后：

```go
c.mu.Lock()
c.counters[name]++
c.mu.Unlock()
```

同一时刻只有一个 goroutine 能修改 map，其他 goroutine 在 `Lock` 处等待。

## 锁的粒度

锁的范围要足够大，能覆盖完整的临界区；但也不要过大，否则会降低并发。

不好的做法：

```go
c.mu.Lock()
defer c.mu.Unlock()

time.Sleep(time.Second) // 持锁做慢操作
c.counters[name]++
```

锁应该只保护共享数据，不要在持锁期间做网络请求、文件 I/O 或长时间计算。

## 不能复制 Mutex

`sync.Mutex` 使用后不能复制：

```go
func bad(c Container) { // 复制了整个 Container，包括锁
	c.inc("a")
}
```

复制后两个锁状态独立，无法保护同一份数据。包含 Mutex 的结构体应通过指针传递：

```go
func good(c *Container) {
	c.inc("a")
}
```

`go vet` 会提示复制包含锁的结构体。

## `RWMutex`：读多写少

如果读操作远多于写操作，可以使用 `sync.RWMutex`：

```go
type Cache struct {
	mu sync.RWMutex
	m  map[string]string
}

func (c *Cache) Get(k string) string {
	c.mu.RLock()
	defer c.mu.RUnlock()
	return c.m[k]
}

func (c *Cache) Set(k, v string) {
	c.mu.Lock()
	defer c.mu.Unlock()
	c.m[k] = v
}
```

- `RLock` 允许多个读者同时进入；
- `Lock` 是写锁，排斥所有读者和写者；
- 读多写少时吞吐量更好。

但 RWMutex 比普通 Mutex 更复杂，读操作很少时不一定更快。先用普通 Mutex，确有性能需求再换。

## 死锁

常见死锁场景：

### 1. 同一 goroutine 重复加锁

```go
mu.Lock()
mu.Lock() // 永久阻塞
```

Go 的 Mutex 不可重入。

### 2. 两个锁顺序不一致

```go
// goroutine A
a.Lock()
b.Lock()

// goroutine B
b.Lock()
a.Lock()
```

两个 goroutine 各持有一把锁，等待对方释放。应统一加锁顺序。

### 3. 忘记解锁

```go
mu.Lock()
// 提前 return，没有 Unlock
```

使用 `defer mu.Unlock()`。

### 4. 解锁未加锁的 Mutex

```go
mu.Unlock()
// fatal error: sync: unlock of unlocked mutex
```

每个 `Lock` 必须对应一个 `Unlock`。

## Mutex、Channel 与 Atomic 如何选择

- **Mutex**：保护共享数据结构，临界区可能包含多条语句；
- **Channel**：传递数据所有权、协调 goroutine、实现流水线；
- **Atomic**：单个变量的简单读改写，例如计数器、标志位。

Go 的格言是“通过通信共享内存”，但这不意味着禁用 Mutex。共享状态确实需要原地修改时，Mutex 往往更直接。

## 常见错误

### 1. 忘记加锁

并发读写 map 会触发 `concurrent map writes` 或数据竞争。

### 2. 锁范围太小

只锁住一部分操作，仍可能读到中间状态。确保整个临界区都被保护。

### 3. 锁范围太大

持锁做 I/O 或长时间计算，导致其他 goroutine 长时间等待。

### 4. 复制包含锁的结构体

锁状态被复制，保护失效。传指针。

### 5. 重复加锁

Mutex 不可重入，同一 goroutine 重复 `Lock` 会死锁。

### 6. 忘记 `defer Unlock`

提前返回或 panic 时锁无法释放。养成 `defer` 习惯。

## 和 Java 对照

| 概念 | Go | Java |
| --- | --- | --- |
| 互斥锁 | `sync.Mutex` | `synchronized` / `ReentrantLock` |
| 读写锁 | `sync.RWMutex` | `ReentrantReadWriteLock` |
| 可重入 | 否 | `synchronized` / `ReentrantLock` 可重入 |
| 条件变量 | `sync.Cond` | `Condition` |
| 并发 map | 手动加锁 / `sync.Map` | `ConcurrentHashMap` |
| 复制 | 禁止 | 对象引用 |

Java 的 `synchronized` 是可重入的，Go 的 Mutex 不是；Java 有 `ConcurrentHashMap` 等并发容器，Go 标准库的普通 map 需要自己加锁。

## 动手练习

1. 用 Mutex 保护一个 map，启动多个 goroutine 并发更新。
2. 去掉锁，运行 `-race`，观察数据竞争。
3. 用 `defer Unlock` 改写，确保异常路径也能解锁。
4. 用 RWMutex 实现一个读多写少的缓存。
5. 构造两个锁顺序相反的死锁，观察程序卡住。
6. 比较 Mutex、atomic 和 channel 实现计数器的差异。

## 小结

- Mutex 保护临界区，同一时刻只允许一个 goroutine 进入。
- 零值 Mutex 可直接使用，使用后不能复制。
- `Lock` 和 `Unlock` 必须配对，优先使用 `defer Unlock`。
- 锁范围要覆盖完整临界区，但不要持锁做慢操作。
- RWMutex 适合读多写少，普通 Mutex 更简单。
- Mutex、channel、atomic 各有适用场景，不要教条选择。

---

上一篇：[[Go/Concurrency/Atomic Counters|Go 并发：原子计数器]]
下一篇：[Go by Example: Stateful Goroutines](https://gobyexample.com/stateful-goroutines)

返回 [[Go/Concurrency/index|Go 并发]]。
