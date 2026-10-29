---
title: "Go 进阶：函数"
description: "从 Go by Example 的 Functions 示例出发，掌握函数声明、参数简写、显式返回、一等函数与 Go 函数设计的常见限制。"
tags:
  - Go
  - 编程语言
  - 进阶
  - 函数
created: 2026-10-29
modified: 2026-10-29
---

# Go 进阶：函数

函数是 Go 程序的基本组成单位。和 Java 不同，Go 的函数可以直接定义在包级别，不需要挂在类上；函数也是一等值，可以赋值、传参和返回。

[Go by Example 的 Functions](https://gobyexample.com/functions) 从两个最简单的求和函数开始。本文补充函数值、值传递、导出规则，以及 Go 有意省略的那些函数特性。

## 完整示例

```go
package main

import "fmt"

func plus(a int, b int) int {
	return a + b
}

func plusPlus(a, b, c int) int {
	return a + b + c
}

func main() {
	res := plus(1, 2)
	fmt.Println("1+2 =", res)

	res = plusPlus(1, 2, 3)
	fmt.Println("1+2+3 =", res)
}
```

运行：

```bash
$ go run functions.go
1+2 = 3
1+2+3 = 6
```

## 声明与调用

函数声明的基本结构：

```go
func 函数名(参数列表) 返回值类型 {
	函数体
}
```

例如：

```go
func plus(a int, b int) int {
	return a + b
}
```

调用时按位置传参：

```go
res := plus(1, 2)
```

Go 的函数名遵循导出规则：首字母大写可以在其他包中调用，小写只在当前包内可见。

函数可以定义在包级别，声明顺序不影响调用。也就是说，`main` 可以调用写在文件后面的函数。

## 参数类型简写

连续多个参数类型相同时，可以只在最后一个参数写类型：

```go
func plusPlus(a, b, c int) int {
	return a + b + c
}
```

这等价于：

```go
func plusPlus(a int, b int, c int) int
```

参数类型不同时不能省略：

```go
// 类型不同时必须分别写
func pair(name string, age int) {
}
```

返回值也可以有多个，下一篇会专门讲。

## 必须显式返回

Go 不会自动返回函数体最后一条表达式的值：

```go
func plus(a int, b int) int {
	a + b
}
// missing return
```

必须写 `return`：

```go
func plus(a int, b int) int {
	return a + b
}
```

有返回值但没有 `return`、`return` 的值数量或类型不匹配，都会在编译期报错。

## 函数是一等值

函数可以赋值给变量、作为参数传递、作为返回值：

```go
func plus(a, b int) int {
	return a + b
}

func main() {
	add := plus
	fmt.Println(add(1, 2)) // 3

	apply := func(f func(int, int) int, a, b int) int {
		return f(a, b)
	}
	fmt.Println(apply(plus, 3, 4)) // 7
}
```

函数类型写成 `func(参数类型) 返回值类型`。匿名函数和闭包会在后续文章展开。

## 参数是值传递

Go 的函数参数按值传递。传入一个变量时，函数拿到的是副本：

```go
func zero(x int) {
	x = 0
}

func main() {
	n := 1
	zero(n)
	fmt.Println(n) // 1
}
```

结构体也会整体复制。如果需要在函数内修改原值，应传指针：

```go
func zeroPtr(x *int) {
	*x = 0
}

zeroPtr(&n)
fmt.Println(n) // 0
```

切片、map、channel 等类型的值包含对底层数据的引用，按值传递时复制的是描述符，底层数据仍然共享。这个差异在切片和 map 的文章里已经详细讨论。

## Go 没有的函数特性

Go 的函数设计非常克制，下面这些在其他语言中常见的特性都不存在：

- **没有默认参数**：调用时必须提供所有参数。
- **没有函数重载**：同一个包内不能定义同名但参数不同的函数。
- **没有命名参数**：调用时不能写 `plus(a: 1, b: 2)`。
- **没有可选参数**：需要可选行为时，使用结构体配置、函数选项模式或变参。

例如，想要“默认值”，通常写成：

```go
func greet(name string) string {
	if name == "" {
		name = "guest"
	}
	return "hello, " + name
}
```

或者用配置结构体：

```go
type Options struct {
	Timeout int
	Retries int
}

func connect(addr string, opts Options) error {
	// ...
	return nil
}
```

## 常见错误

### 1. 缺少 `return`

有返回值的函数必须保证所有路径都有返回值。编译器会检查。

### 2. 返回值类型不匹配

`return "1"` 不能作为 `int` 返回值。Go 不会自动把字符串转成数字。

### 3. 参数数量或类型错误

Go 不支持重载，也没有默认参数，调用时参数必须完全匹配。

### 4. 试图定义同名重载函数

```go
func add(a, b int) int
func add(a, b, c int) int
// add redeclared in this block
```

需要不同行为时，使用不同的函数名，或使用变参/配置结构体。

### 5. 以为参数是引用传递

普通参数是副本。修改参数不会影响调用方，除非参数本身包含共享的底层数据，或者你传的是指针。

## 和 Java 对照

| 概念 | Go | Java |
| --- | --- | --- |
| 函数位置 | 包级别，可直接定义 | 必须定义在类/接口中 |
| 函数重载 | 不支持 | 支持 |
| 默认参数 | 不支持 | 不支持（常用重载模拟） |
| 命名参数 | 不支持 | 不支持 |
| 一等函数 | `func` 值 | Lambda / 方法引用 / 函数式接口 |
| 参数传递 | 值传递 | 值传递（对象传递引用副本） |
| 导出规则 | 首字母大写 | `public` 等访问修饰符 |

Java 的方法必须属于某个类，Go 的函数可以直接放在包里。Java 通过重载提供多种参数组合，Go 更倾向于用清晰的函数名或配置结构体表达。

## 动手练习

1. 写一个 `max(a, b int) int` 函数，返回较大的值。
2. 用参数类型简写定义三个 `int` 参数的求和函数。
3. 把函数赋值给变量，再通过变量调用。
4. 写一个接收函数参数的 `apply` 函数，分别传入加法和乘法。
5. 写一个修改 `int` 参数的函数，观察调用方是否变化，再用指针改写。
6. 尝试定义同名重载函数，观察编译错误。

## 小结

- 函数用 `func` 声明，参数和返回值类型必须明确。
- 连续同类型参数可以简写类型。
- Go 要求显式 `return`，不会自动返回最后一条表达式。
- 函数是一等值，可以赋值、传参和返回。
- 参数按值传递，修改原值需要指针。
- Go 没有默认参数、函数重载和命名参数。

---

上一篇：[[Go/Basics/Maps|Go 基础：map]]
下一篇：[[Go/Advanced/Multiple Return Values|Go 进阶：多返回值]]

返回 [[Go/Advanced/index|Go 进阶]]。
