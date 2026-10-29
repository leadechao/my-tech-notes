---
title: "Go 进阶：range 遍历迭代器"
description: "从 Go by Example 的 Range over Iterators 示例出发，理解 Go 1.23 的 iter.Seq、yield、惰性迭代、无限序列与标准库迭代器工具。"
tags:
  - Go
  - 编程语言
  - 进阶
  - 迭代器
created: 2026-10-29
modified: 2026-10-29
---

# Go 进阶：range 遍历迭代器

Go 1.23 引入了迭代器（iterator）和 `range` over function。现在，只要有合适的函数签名，自定义类型、无限序列、流式数据都可以直接放进 `for range`。

这改变了 Go 遍历自定义数据结构的方式：不再需要先构造一个切片，也不需要手写 `Next()` 循环。

[Go by Example 的 Range over Iterators](https://gobyexample.com/range-over-iterators) 用链表和斐波那契演示了迭代器。本文用更简单的切片和无限序列说明核心机制。

## 完整示例

```go
package main

import (
	"fmt"
	"iter"
	"slices"
	"strings"
)

func All[T any](xs []T) iter.Seq[T] {
	return func(yield func(T) bool) {
		for _, x := range xs {
			if !yield(x) {
				return
			}
		}
	}
}

func genFib() iter.Seq[int] {
	return func(yield func(int) bool) {
		a, b := 0, 1
		for {
			if !yield(a) {
				return
			}
			a, b = b, a+b
		}
	}
}

func main() {
	for x := range All([]int{10, 13, 23}) {
		fmt.Println(x)
	}

	all := slices.Collect(All([]int{10, 13, 23}))
	fmt.Println("all:", all)

	for part := range strings.SplitSeq("go-by-example", "-") {
		fmt.Printf("part: %s\n", part)
	}

	for n := range genFib() {
		if n >= 10 {
			break
		}
		fmt.Println(n)
	}
}
```

运行：

```bash
$ go run range-over-iterators.go
10
13
23
all: [10 13 23]
part: go
part: by
part: example
0
1
1
2
3
5
8
```

## 迭代器的本质

`iter.Seq[T]` 的定义是：

```go
type Seq[T any] func(yield func(T) bool)
```

它是一个函数，接收一个 `yield` 回调。迭代器每产生一个元素，就调用一次 `yield`：

```go
func All[T any](xs []T) iter.Seq[T] {
	return func(yield func(T) bool) {
		for _, x := range xs {
			if !yield(x) {
				return
			}
		}
	}
}
```

`yield` 返回 `bool`：

- `true`：调用方还想继续；
- `false`：调用方提前退出（`break`、`return` 等），迭代器应立即停止。

所以迭代器内部必须检查 `yield` 的返回值，否则提前退出后还会继续计算，造成浪费甚至副作用。

## 用 `range` 消费迭代器

```go
for x := range All([]int{10, 13, 23}) {
	fmt.Println(x)
}
```

`range` 会自动调用迭代器函数，并把一个内部 `yield` 传进去。循环体每执行一次，就对应一次 `yield`。

当循环 `break` 或函数 `return` 时，`yield` 返回 `false`，迭代器收到信号后停止。

## 无限序列

迭代器不要求有底层数据结构，也不要求有限：

```go
func genFib() iter.Seq[int] {
	return func(yield func(int) bool) {
		a, b := 0, 1
		for {
			if !yield(a) {
				return
			}
			a, b = b, a+b
		}
	}
}
```

消费时必须提供退出条件：

```go
for n := range genFib() {
	if n >= 10 {
		break
	}
	fmt.Println(n)
}
```

如果既不 `break` 也不 `return`，这个循环永远不会结束。惰性迭代把“何时停止”的控制权交给了调用方。

## 标准库中的迭代器

### `slices.Collect`

`Collect` 把任意迭代器收集成切片：

```go
all := slices.Collect(All([]int{10, 13, 23}))
fmt.Println(all) // [10 13 23]
```

### `slices.Values`

`Values` 把切片转成迭代器：

```go
for x := range slices.Values([]int{1, 2, 3}) {
	fmt.Println(x)
}
```

### `strings.SplitSeq`

`SplitSeq` 按分隔符流式切分字符串，不先构造结果切片：

```go
for part := range strings.SplitSeq("go-by-example", "-") {
	fmt.Println(part)
}
```

对很大的字符串，这能减少一次性的内存分配。

### `maps.All`

`maps.All` 遍历 map 的键值对：

```go
for k, v := range maps.All(m) {
	fmt.Println(k, v)
}
```

它返回 `iter.Seq2[K, V]`，对应 `range` 的两个变量。

## `iter.Seq2`：两个返回值

`iter.Seq[T]` 每次产出一个值。需要键值对时使用 `iter.Seq2[K, V]`：

```go
type Seq2[K, V any] func(yield func(K, V) bool)
```

`maps.All`、`slices.Backward` 等函数会返回 `Seq2`。`range` 的写法不变：

```go
for k, v := range seq2 {
	// ...
}
```

## 自定义类型实现迭代器

给类型添加一个返回 `iter.Seq[T]` 的方法，就可以让它支持 `range`：

```go
type List[T any] struct {
	items []T
}

func (l *List[T]) All() iter.Seq[T] {
	return func(yield func(T) bool) {
		for _, item := range l.items {
			if !yield(item) {
				return
			}
		}
	}
}
```

使用：

```go
lst := &List[int]{items: []int{10, 13, 23}}
for x := range lst.All() {
	fmt.Println(x)
}
```

这就是 Go by Example 中 `List.All` 的思路。相比返回 `[]T`，迭代器可以惰性产生元素，也可以表示无限序列。

## 常见错误

### 1. 忽略 `yield` 的返回值

```go
func All[T any](xs []T) iter.Seq[T] {
	return func(yield func(T) bool) {
		for _, x := range xs {
			yield(x) // 错误：没有检查提前退出
		}
	}
}
```

调用方 `break` 后，迭代器仍会继续执行。应在 `yield` 返回 `false` 时立即 `return`。

### 2. 在 Go 1.22 及更早版本使用

`iter` 包和 `range` over function 需要 Go 1.23+。旧版本无法编译。

### 3. 忘记退出条件

无限迭代器必须由调用方 `break` 或 `return`，否则程序不会结束。

### 4. 在迭代器里启动 goroutine 却不清理

如果迭代器内部启动后台任务，提前退出时需要通过 `yield` 的返回值或 `defer` 做清理，避免 goroutine 泄漏。

### 5. 误以为迭代器是“拉”模型

Go 的迭代器是“推”模型：迭代器主动调用 `yield`。它不像 Java 的 `Iterator.next()` 由调用方拉取。不要按拉模型写控制流。

## 和 Java 对照

| 概念 | Go | Java |
| --- | --- | --- |
| 迭代方式 | 推送式 `yield` | 拉取式 `Iterator.next()` |
| 核心类型 | `iter.Seq[T]` | `Iterable<T>` / `Iterator<T>` |
| 两个值 | `iter.Seq2[K,V]` | `Map.Entry` |
| 流式处理 | `strings.SplitSeq` | `Stream` / `Spliterator` |
| 无限序列 | 支持 | `Stream.iterate` 等 |
| 提前终止 | `yield` 返回 false | `break` / `hasNext` |

Java 的 `Iterator` 是拉模型，调用方决定何时 `next()`；Go 的迭代器是推模型，迭代器决定何时调用 `yield`。两者都能实现惰性求值，但控制流方向相反。

## 动手练习

1. 写一个 `All[T]` 迭代器，遍历切片并支持 `break`。
2. 用 `slices.Collect` 把迭代器结果收集成切片。
3. 写一个无限自然数迭代器，只取前 10 个。
4. 用 `strings.SplitSeq` 流式切分一段文本。
5. 给自定义 `List[T]` 添加 `All()` 方法，并用 `range` 遍历。
6. 故意不检查 `yield` 返回值，观察 `break` 后迭代器是否继续执行。

## 小结

- Go 1.23 引入 `iter.Seq[T]`，本质是 `func(yield func(T) bool)`。
- `range` 可以消费迭代器，`break` 会让 `yield` 返回 `false`。
- 迭代器可以是惰性的、无限的，不要求底层数据结构。
- `iter.Seq2` 用于键值对等两个返回值的场景。
- 标准库提供 `slices.Collect`、`slices.Values`、`strings.SplitSeq`、`maps.All` 等工具。
- 迭代器内部必须检查 `yield` 返回值，提前退出时及时停止。

---

上一篇：[[Go/Advanced/Generics|Go 进阶：泛型]]
下一篇：[[Go/Advanced/Errors|Go 进阶：错误处理]]

返回 [[Go/Advanced/index|Go 进阶]]。
