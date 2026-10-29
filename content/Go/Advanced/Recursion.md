---
title: "Go 进阶：递归"
description: "从 Go by Example 的 Recursion 示例出发，理解基准条件、递归调用、匿名递归、调用栈，以及递归的性能与栈溢出风险。"
tags:
  - Go
  - 编程语言
  - 进阶
  - 递归
created: 2026-10-29
modified: 2026-10-29
---

# Go 进阶：递归

递归是函数直接或间接调用自身。它适合处理天然具有层次结构的问题：树、图、分治、回溯。写递归最关键的是两件事：**基准条件**和**向基准条件收敛**。

[Go by Example 的 Recursion](https://gobyexample.com/recursion) 用阶乘和斐波那契演示了递归。本文补充匿名递归、调用栈、性能陷阱和常见错误。

## 完整示例

```go
package main

import "fmt"

func fact(n int) int {
	if n == 0 {
		return 1
	}
	return n * fact(n-1)
}

func main() {
	fmt.Println(fact(7))

	var fib func(n int) int

	fib = func(n int) int {
		if n < 2 {
			return n
		}
		return fib(n-1) + fib(n-2)
	}

	fmt.Println(fib(7))
}
```

运行：

```bash
$ go run recursion.go
5040
13
```

## 基准条件与递归调用

阶乘函数：

```go
func fact(n int) int {
	if n == 0 {
		return 1
	}
	return n * fact(n-1)
}
```

它包含两个部分：

- **基准条件**：`n == 0` 时返回 `1`，不再递归；
- **递归调用**：`fact(n-1)` 把问题规模缩小，最终一定会到达 `0`。

如果没有基准条件，或者递归参数没有向基准条件靠近，就会无限递归，最终栈溢出。

## 匿名函数递归

匿名函数不能直接用自己的名字调用自己，因为函数字面量还没有赋值给变量。需要先声明变量：

```go
var fib func(n int) int

fib = func(n int) int {
	if n < 2 {
		return n
	}
	return fib(n-1) + fib(n-2)
}
```

这里先声明 `fib`，再赋值。匿名函数体里的 `fib` 引用的是已经声明的变量，因此可以递归调用。

不能写成：

```go
fib := func(n int) int {
	return fib(n - 1) // 此时 fib 还没有声明
}
```

`:=` 的右边先求值，函数字面量创建时 `fib` 还不存在，编译器会报 `undefined: fib`。

## 递归与调用栈

每次递归调用都会在调用栈上压入一帧，保存参数、局部变量和返回地址。递归太深会耗尽栈空间：

```text
runtime: goroutine stack exceeds 1000000000-byte limit
fatal error: stack overflow
```

Go 的 goroutine 栈会按需增长，比固定栈的线程更灵活，但增长有上限。处理非常深的结构时，应考虑迭代或显式使用栈。

Go **没有尾调用优化**。即使写成尾递归，编译器也不会自动把它转成循环。需要常量栈空间时，应手写迭代版本。

## 递归的性能陷阱

斐波那契的朴素递归：

```go
func fib(n int) int {
	if n < 2 {
		return n
	}
	return fib(n-1) + fib(n-2)
}
```

时间复杂度是 O(2^n)，因为同一个子问题会被重复计算。`fib(40)` 就会明显变慢，`fib(50)` 可能难以等待。

改进方式：

1. **记忆化**：缓存已经计算过的结果；
2. **迭代**：用两个变量滚动计算；
3. **动态规划**：自底向上填表。

迭代版本：

```go
func fib(n int) int {
	if n < 2 {
		return n
	}
	a, b := 0, 1
	for i := 2; i <= n; i++ {
		a, b = b, a+b
	}
	return b
}
```

阶乘也有类似问题：`fact(21)` 会超出 64 位 `int` 的表示范围，发生溢出。递归本身没错，但要选择合适的数据类型和算法。

## 什么时候用递归

适合递归的场景：

- 树和图的遍历；
- 分治算法（归并排序、快速排序）；
- 回溯（排列、组合、数独）；
- 解析嵌套结构（JSON、表达式、文件系统）。

不适合递归的场景：

- 简单的线性循环；
- 深度可能非常大的链表或数组；
- 子问题大量重复且没有缓存；
- 对栈空间有严格限制的环境。

## 常见错误

### 1. 缺少基准条件

递归永远不会停止，最终栈溢出。

### 2. 递归参数没有收敛

例如 `fact(n)` 调用 `fact(n)`，规模没有缩小。每次递归都应向基准条件靠近。

### 3. 基准条件写错

`fact(0)` 返回 `0` 会导致所有结果变成 `0`。基准条件要单独验证。

### 4. 匿名函数直接自引用

必须先用 `var` 声明函数变量，再赋值。`:=` 无法在函数体里引用自身。

### 5. 以为 Go 会做尾递归优化

Go 不做尾调用优化。深递归需要改成迭代。

### 6. 忽略重复计算

朴素斐波那契是指数级。使用记忆化或迭代。

### 7. 忽略整数溢出

阶乘增长极快，`int64` 也只能容纳到 `20!` 左右。必要时使用 `math/big`。

## 和 Java 对照

| 概念 | Go | Java |
| --- | --- | --- |
| 递归语法 | 直接调用函数 | 直接调用方法 |
| 尾调用优化 | 不支持 | 通常也不支持 |
| 栈 | goroutine 栈动态增长 | 线程栈固定（可配置） |
| 栈溢出 | `fatal error: stack overflow` | `StackOverflowError` |
| 大整数 | `math/big` | `BigInteger` |

Java 的递归受线程栈大小限制，通常更容易触发 `StackOverflowError`；Go 的 goroutine 栈可以增长，但同样有上限。

## 动手练习

1. 写递归阶乘，分别计算 `5!`、`10!`、`20!`。
2. 写递归斐波那契，再改写成迭代版本，比较 `fib(35)` 的耗时。
3. 用记忆化优化斐波那契。
4. 写一个递归函数计算整数各位数字之和。
5. 用匿名函数实现递归斐波那契，注意先声明变量。
6. 写一个递归目录遍历函数，处理嵌套目录。

## 小结

- 递归必须有基准条件，并且每次递归都要向基准条件收敛。
- 匿名函数递归需要先用 `var` 声明函数变量。
- 每次递归都会消耗栈空间，Go 不做尾调用优化。
- 朴素递归可能重复计算，必要时使用记忆化或迭代。
- 递归适合树、分治和回溯，简单线性问题优先用循环。

---

上一篇：[[Go/Advanced/Closures|Go 进阶：闭包]]
下一篇：[[Go/Advanced/Range Over Built-in Types|Go 进阶：range 遍历内置类型]]

返回 [[Go/Advanced/index|Go 进阶]]。
