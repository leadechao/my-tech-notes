---
title: "Go 进阶：变参函数"
description: "从 Go by Example 的 Variadic Functions 示例出发，掌握 ...T 参数、切片展开、零参数调用、定参混合与常见限制。"
tags:
  - Go
  - 编程语言
  - 进阶
  - 函数
created: 2026-10-29
modified: 2026-10-29
---

# Go 进阶：变参函数

变参函数可以接收任意数量的尾部参数。`fmt.Println` 就是最常见的例子：你可以传一个值、多个值，甚至一个切片展开。

[Go by Example 的 Variadic Functions](https://gobyexample.com/variadic-functions) 用求和函数演示了基本用法。本文补充变参的类型本质、切片展开、定参混合和底层共享问题。

## 完整示例

```go
package main

import "fmt"

func sum(nums ...int) {
	fmt.Print(nums, " ")
	total := 0
	for _, num := range nums {
		total += num
	}
	fmt.Println(total)
}

func main() {
	sum(1, 2)
	sum(1, 2, 3)

	nums := []int{1, 2, 3, 4}
	sum(nums...)
}
```

运行：

```bash
$ go run variadic-functions.go
[1 2] 3
[1 2 3] 6
[1 2 3 4] 10
```

## `...T` 参数

```go
func sum(nums ...int) {
	// ...
}
```

`...int` 表示“零个或多个 `int`”。在函数内部，`nums` 的类型就是 `[]int`，可以调用 `len`、`cap`，也可以用 `range` 遍历：

```go
func sum(nums ...int) int {
	total := 0
	for _, n := range nums {
		total += n
	}
	return total
}
```

变参参数看起来像切片，但调用方式不同：调用者可以直接列出多个值，不需要先构造切片。

## 零个参数也可以

变参函数允许不传任何可变参数：

```go
sum() // 输出 [] 0
```

此时 `nums` 是 nil，长度为 0。如果传入一个非 nil 的空切片（例如 `sum([]int{}...)`），`nums` 会是非 nil、长度仍为 0。通常只需要用 `len(nums)` 判断是否有元素，不必依赖 nil 状态。

## 用 `...` 展开切片

如果已经有一个切片，可以在调用时加上 `...`，把它展开成多个参数：

```go
nums := []int{1, 2, 3, 4}
sum(nums...)
```

不写 `...` 会编译失败，因为 `[]int` 不能直接赋给 `...int`：

```go
sum(nums)
// cannot use nums (variable of type []int) as int value in argument to sum
```

`...` 是调用点的语法，表示“把切片元素逐个传入”。

## 定参与变参混合

变参参数必须是最后一个参数，前面可以有普通参数：

```go
func greet(prefix string, names ...string) {
	for _, name := range names {
		fmt.Println(prefix, name)
	}
}

greet("hello", "Alice", "Bob")
```

不能把变参参数放在中间：

```go
// func bad(nums ...int, name string) // 语法错误
```

一个函数也只能有一个变参参数。

调用带定参的变参函数时，`...` 仍必须放在最后：

```go
prefixes := []string{"hi", "hey"}
// greet(prefixes..., "Alice") // 错误：... 不是最后一个参数
```

如果定参也需要从切片传入，只能逐个取，或者重新设计函数签名。

## 变参与切片共享底层数组

当使用 `slice...` 调用变参函数时，函数内部的 `nums` 与原切片共享底层数组：

```go
func modify(nums ...int) {
	nums[0] = 99
}

func main() {
	a := []int{1, 2}
	modify(a...)
	fmt.Println(a) // [99 2]
}
```

如果函数要修改参数而不影响调用方，应先复制一份：

```go
func modify(nums ...int) {
	local := append([]int(nil), nums...)
	local[0] = 99
}
```

直接传多个值调用时，编译器会创建一个新的切片，通常不存在这个问题；但不要依赖实现细节，明确复制最安全。

## 常见错误

### 1. 传入切片时忘记 `...`

```go
sum(nums) // 错误
sum(nums...) // 正确
```

### 2. 变参参数不在最后

Go 要求 `...T` 是最后一个参数。需要“可选的其他参数”时，可以把它们放进结构体。

### 3. 试图定义多个变参参数

一个函数只能有一个 `...T` 参数。

### 4. `...` 后面还有其他参数

```go
sum(nums..., 1) // 错误
```

`...` 只能出现在调用参数列表的末尾。

### 5. 以为变参一定会复制

使用 `slice...` 时，函数拿到的是同一个底层数组的切片视图。修改元素可能影响调用方。

### 6. 用 nil 状态判断是否有元素

无参数调用时 `nums` 是 nil；传入非 nil 空切片时，`nums` 不是 nil。两者长度都是 0，业务上通常应使用 `len(nums)` 判断是否有元素，而不是 `nums == nil`。

## 和 Java 对照

| 概念 | Go | Java |
| --- | --- | --- |
| 语法 | `func f(nums ...int)` | `void f(int... nums)` |
| 内部类型 | `[]int` | `int[]` |
| 位置 | 必须是最后一个参数 | 必须是最后一个参数 |
| 传数组/切片 | `f(slice...)` | `f(array)` |
| 零参数 | 允许 | 允许 |
| 多个变参 | 不允许 | 不允许 |

Java 的可变参数本质上也是数组，调用时直接传数组即可；Go 需要显式写 `...` 来展开切片。这个差异在把 Java 代码迁移到 Go 时很容易遗漏。

## 动手练习

1. 写一个 `sum(nums ...int) int`，分别用零个、两个、多个参数调用。
2. 用切片加 `...` 调用上面的函数。
3. 写一个 `greet(prefix string, names ...string)`，混合定参和变参。
4. 在变参函数中修改 `nums[0]`，观察调用方切片是否变化。
5. 尝试把变参参数放在参数列表中间，观察编译错误。
6. 尝试调用 `sum(nums..., 1)`，观察错误信息。

## 小结

- `...T` 表示零个或多个 `T`，在函数内部就是 `[]T`。
- 变参参数必须是最后一个参数，一个函数只能有一个。
- 调用时可以用 `slice...` 展开切片。
- 使用 `slice...` 时，函数内部与原切片共享底层数组。
- 需要独立数据时，先复制切片。

---

上一篇：[[Go/Advanced/Multiple Return Values|Go 进阶：多返回值]]
下一篇：[[Go/Advanced/Closures|Go 进阶：闭包]]

返回 [[Go/Advanced/index|Go 进阶]]。
