---
title: "Go 进阶：闭包"
description: "从 Go by Example 的 Closures 示例出发，理解匿名函数、变量捕获、独立状态、循环变量捕获与闭包的常见用途。"
tags:
  - Go
  - 编程语言
  - 进阶
  - 闭包
created: 2026-10-29
modified: 2026-10-29
---

# Go 进阶：闭包

闭包是“函数 + 它捕获的外部变量”。Go 的函数是一等值，可以在函数内部定义匿名函数，并让它记住外层作用域中的变量。

[Go by Example 的 Closures](https://gobyexample.com/closures) 用计数器演示了闭包如何保存状态。本文补充引用捕获、独立状态、循环变量陷阱和实际用途。

## 完整示例

```go
package main

import "fmt"

func intSeq() func() int {
	i := 0
	return func() int {
		i++
		return i
	}
}

func main() {
	nextInt := intSeq()

	fmt.Println(nextInt())
	fmt.Println(nextInt())
	fmt.Println(nextInt())

	newInts := intSeq()
	fmt.Println(newInts())
}
```

运行：

```bash
$ go run closures.go
1
2
3
1
```

## 匿名函数

没有名字的函数叫匿名函数：

```go
func() {
	fmt.Println("hello")
}()
```

末尾的 `()` 表示立即调用。匿名函数也可以赋值给变量：

```go
greet := func(name string) {
	fmt.Println("hello", name)
}
greet("Go")
```

匿名函数适合只在一个地方使用的小逻辑，例如回调、过滤条件和测试辅助函数。

## 闭包捕获外部变量

`intSeq` 返回一个匿名函数，这个匿名函数引用了外层的局部变量 `i`：

```go
func intSeq() func() int {
	i := 0
	return func() int {
		i++
		return i
	}
}
```

`i` 本来应该在 `intSeq` 返回后消失，但闭包仍然持有它，所以每次调用 `nextInt` 都能在上一次的基础上递增。这就是闭包保存状态的方式。

关键点是：闭包捕获的是**变量本身**，不是调用时的值副本。闭包对 `i` 的修改会持续保留。

## 每次调用得到独立状态

每调用一次 `intSeq()`，都会创建一个新的 `i`：

```go
nextInt := intSeq()
newInts := intSeq()

fmt.Println(nextInt()) // 1
fmt.Println(nextInt()) // 2
fmt.Println(newInts()) // 1
```

`nextInt` 和 `newInts` 捕获的是两个不同的 `i`。如果把它们赋给同一个函数变量，则会共享状态：

```go
a := intSeq()
b := a

fmt.Println(a()) // 1
fmt.Println(b()) // 2
```

这里 `a` 和 `b` 是同一个函数值，底层捕获的是同一个 `i`。

## 闭包可以捕获多个变量

```go
func counter(start int) (func() int, func()) {
	count := start
	return func() int {
			count++
			return count
		}, func() {
			count = start
		}
}
```

两个返回的函数共享同一个 `count`，一个负责递增，一个负责重置。这种模式可以实现简单的状态机或有状态回调。

## 循环变量捕获

闭包和循环一起使用时，要特别注意循环变量的作用域。

Go 1.22 起，`for` 循环变量在每次迭代中都是新变量：

```go
var funcs []func()
for i := 0; i < 3; i++ {
	funcs = append(funcs, func() {
		fmt.Println(i)
	})
}
for _, f := range funcs {
	f()
}
```

输出是 `0 1 2`，每个闭包捕获各自的 `i`。

在 Go 1.21 及更早版本中，所有闭包会共享同一个循环变量，最后通常都打印 `3`。如果项目需要兼容旧版本，应在循环体内复制一份：

```go
for i := 0; i < 3; i++ {
	i := i
	funcs = append(funcs, func() {
		fmt.Println(i)
	})
}
```

> [!note]
> 闭包捕获的是变量，不是值。理解这一点，就能解释“为什么状态会保留”和“为什么循环变量会串”这两个常见现象。

## 闭包的常见用途

### 1. 有状态函数

计数器、生成器、限流器都可以用闭包实现：

```go
func nextID() func() int {
	id := 0
	return func() int {
		id++
		return id
	}
}
```

### 2. 回调与中间件

```go
func withLogging(fn func(string)) func(string) {
	return func(s string) {
		fmt.Println("before:", s)
		fn(s)
		fmt.Println("after:", s)
	}
}
```

### 3. 自定义排序

```go
sort.Slice(users, func(i, j int) bool {
	return users[i].Age < users[j].Age
})
```

比较函数捕获了 `users`，在排序过程中反复使用。

### 4. 延迟执行

```go
func() {
	defer func() {
		fmt.Println("cleanup")
	}()
	// ...
}()
```

`defer` 后面的匿名函数会在函数返回时执行。

## 闭包与内存

闭包会让被捕获的变量延长生命周期。如果闭包逃逸到函数外，这些变量通常也会逃逸到堆上，直到闭包不再被引用为止。大量创建闭包时要注意内存占用，尤其是捕获了大对象或切片的情况。

闭包本身是函数值，通常很小；真正占内存的是它引用的外部数据。

## 常见错误

### 1. 以为闭包捕获的是值副本

闭包捕获变量本身。多个闭包共享同一个变量时，会看到彼此的修改。

### 2. 在旧版本 Go 中捕获循环变量

Go 1.21 及更早版本中，循环变量在所有迭代间共享。需要在循环体内复制。

### 3. 调用 nil 函数值

```go
var f func()
f()
// panic: runtime error: invalid memory address or nil pointer dereference
```

声明函数变量后要赋值再调用。

### 4. 闭包持有大对象导致内存无法释放

闭包会延长被捕获变量的生命周期。不需要时，避免捕获不必要的大对象。

### 5. 在并发场景中共享闭包状态

闭包捕获的变量不是并发安全的。多个 goroutine 同时调用同一个闭包时，需要加锁或使用原子操作。

## 和 Java 对照

| 概念 | Go | Java |
| --- | --- | --- |
| 匿名函数 | `func(...) {...}` | Lambda `(x) -> ...` |
| 捕获局部变量 | 可捕获并修改 | 只能捕获 effectively final 变量 |
| 函数类型 | 原生 `func` 类型 | 函数式接口 |
| 状态保存 | 闭包直接持有变量 | 需要对象字段或数组等可变容器 |
| 循环变量 | Go 1.22 起每次迭代新变量 | Lambda 捕获 effectively final 副本 |

Java 的 Lambda 不能修改捕获的局部变量，因此有状态逻辑通常要借助对象字段；Go 的闭包可以直接修改捕获的变量，写起来更灵活，但也更容易产生共享状态问题。

## 动手练习

1. 写一个 `intSeq` 闭包，连续调用输出 `1、2、3`。
2. 创建两个独立的计数器，验证它们的状态互不影响。
3. 写一个带 `start` 参数的计数器，并返回递增和重置两个闭包。
4. 用闭包实现一个简单的限流器：每调用 N 次执行一次回调。
5. 在 Go 1.22+ 中写循环闭包，观察输出；再用局部变量复制的方式改写。
6. 用 `sort.Slice` 和闭包按年龄排序一组结构体。

## 小结

- 匿名函数是没有名字的函数，可以立即调用或赋值给变量。
- 闭包捕获外部变量本身，因此可以保存和修改状态。
- 每次调用外层函数都会创建新的捕获变量，状态相互独立。
- Go 1.22 起循环变量每次迭代都是新变量，闭包捕获更符合直觉。
- 闭包适合有状态函数、回调、中间件和自定义排序，但要注意共享状态和内存生命周期。

---

上一篇：[[Go/Advanced/Variadic Functions|Go 进阶：变参函数]]
下一篇：[[Go/Advanced/Recursion|Go 进阶：递归]]

返回 [[Go/Advanced/index|Go 进阶]]。
