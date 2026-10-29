---
title: "Go 基础：for 循环与 range"
description: "从 Go by Example 的 For 示例出发，掌握 Go 唯一的循环结构：条件循环、经典三段式、range 整数、无限循环、break/continue 与标签。"
tags:
  - Go
  - 编程语言
  - 基础
  - 循环
created: 2026-10-29
modified: 2026-10-29
---

# Go 基础：for 循环与 range

Go 只有一个循环关键字：`for`。没有 `while`，也没有 `do-while`。看起来少了语法糖，实际上所有循环都能用 `for` 的几种形式表达。

[Go by Example 的 For](https://gobyexample.com/for) 展示了五种常见写法。本文补充标签跳转、字符串遍历、循环变量语义和常见错误。

## 完整示例

```go
package main

import "fmt"

func main() {
	i := 1
	for i <= 3 {
		fmt.Println(i)
		i = i + 1
	}

	for j := 0; j < 3; j++ {
		fmt.Println(j)
	}

	for i := range 3 {
		fmt.Println("range", i)
	}

	for {
		fmt.Println("loop")
		break
	}

	for n := range 6 {
		if n%2 == 0 {
			continue
		}
		fmt.Println(n)
	}
}
```

运行：

```bash
$ go run for.go
1
2
3
0
1
2
range 0
range 1
range 2
loop
1
3
5
```

## 只有条件的循环

最简单的形式只写条件，等价于其他语言里的 `while`：

```go
i := 1
for i <= 3 {
	fmt.Println(i)
	i = i + 1
}
```

条件必须是 `bool`，Go 不会把整数或字符串自动转换成布尔值：

```go
for 1 {
}
// non-boolean condition in for statement
```

循环体必须用花括号包起来。即使只有一行语句，也不能省略。

## 经典三段式

初始化、条件、后置语句用分号分隔：

```go
for j := 0; j < 3; j++ {
	fmt.Println(j)
}
```

三个部分都可以省略：

```go
sum := 0
for i := 0; i < 10; i++ {
	sum += i
}

// 只保留条件
for sum < 100 {
	sum += 10
}

// 无限循环
for {
}
```

`i++` 是语句，不是表达式，不能写成 `x = i++`。Go 只支持后缀 `i++` 作为语句，不支持前缀 `++i`。

## 用 `range` 遍历整数

Go 1.22 起，可以直接对整数使用 `range`：

```go
for i := range 3 {
	fmt.Println("range", i)
}
// range 0
// range 1
// range 2
```

它等价于遍历 `0` 到 `n-1`，不包括 `n`。如果项目需要兼容 Go 1.21 及更早版本，应写成经典三段式：

```go
for i := 0; i < 3; i++ {
	fmt.Println("range", i)
}
```

## 无限循环与退出

省略条件的 `for` 会一直执行，直到遇到 `break` 或函数返回：

```go
for {
	fmt.Println("loop")
	break
}
```

实际代码中常见的写法是配合退出条件：

```go
for {
	line, err := reader.ReadString('\n')
	if err != nil {
		break
	}
	fmt.Print(line)
}
```

`break` 跳出当前循环，`continue` 跳过本次循环剩余的语句，直接进入下一次迭代。在三段式循环中，`continue` 之后仍会执行后置语句（例如 `i++`）。

## 标签：跳出多层循环

`break` 和 `continue` 默认只作用于最内层循环。嵌套循环需要跳出外层时，可以使用标签：

```go
outer:
for i := 0; i < 3; i++ {
	for j := 0; j < 3; j++ {
		if j == 1 {
			continue outer
		}
		fmt.Println(i, j)
	}
}
```

输出：

```text
0 0
1 0
2 0
```

`continue outer` 会跳过内层循环剩余部分，直接进入外层循环的下一次迭代。把 `continue` 换成 `break outer`，则会完全跳出外层循环。

标签必须紧挨着 `for`、`switch` 或 `select` 语句，并且只在当前函数内有效。

## `range` 不止能遍历整数

`range` 最常见的用途是遍历字符串、数组、切片、map 和 channel。以字符串为例：

```go
for i, r := range "Go语言" {
	fmt.Println(i, string(r))
}
```

输出：

```text
0 G
1 o
2 语
5 言
```

第一个返回值是字节索引，第二个返回值是 `rune`。因为一个中文字符在 UTF-8 中占 3 个字节，所以 `"语"` 后面的索引直接跳到了 `5`。

如果只需要索引，可以省略第二个变量；如果只需要值，可以用 `_` 忽略索引：

```go
for _, r := range "Go语言" {
	fmt.Println(string(r))
}
```

数组、切片和 map 的遍历会在后续文章中展开。map 的遍历顺序是随机的，不能依赖。

> [!note]
> Go 1.22 起，`for` 循环变量在每次迭代中都是新变量。闭包捕获循环变量时，不会再出现所有闭包共享同一个变量的问题。升级旧代码时，可以留意这个行为变化。

## 常见错误

### 1. 以为有 `while`

Go 没有 `while` 关键字。用 `for condition { ... }` 代替。

### 2. 循环条件不是 `bool`

`for 1 {}`、`for "yes" {}` 都会编译失败。条件必须是布尔表达式。

### 3. 在循环外使用 `break`

`break` 只能出现在 `for`、`switch`、`select` 中。单独写 `break` 会报 `break is not in a loop, switch, or select`。

### 4. 在 Go 1.21 及更早版本使用 `range` 整数

`for i := range 3` 需要 Go 1.22+。旧版本请使用三段式循环。

### 5. 忘记推进循环变量

```go
i := 0
for i < 3 {
	fmt.Println(i)
}
```

这个循环永远不会结束。条件循环需要自己在循环体里更新变量。

### 6. `break` 只跳出内层循环

嵌套循环中想跳出外层，需要使用标签，或者让内层循环返回一个标志。

## 和 Java 对照

| 概念 | Go | Java |
| --- | --- | --- |
| 循环关键字 | 只有 `for` | `for`、`while`、`do-while` |
| 条件循环 | `for cond { }` | `while (cond) { }` |
| 经典循环 | `for i := 0; i < n; i++` | `for (int i = 0; i < n; i++)` |
| 花括号 | 必须 | 单条语句可省略 |
| 增强 for | `for i, v := range xs` | `for (var v : xs)` |
| 标签跳转 | 支持 `break label` / `continue label` | 支持带标签的 `break` / `continue` |
| 循环变量捕获 | Go 1.22 起每次迭代新变量 | 每次迭代新变量（传统 for 需注意） |

Java 的 `for` 头部必须用括号包住，Go 不用括号，但三段式内部的分号不能省略。

## 动手练习

1. 用条件循环打印 `1` 到 `10`。
2. 用经典三段式计算 `1` 到 `100` 的和。
3. 用 `range` 打印 `0` 到 `4`，再用 `continue` 跳过奇数。
4. 写一个嵌套循环，用 `break outer` 在内层满足条件时跳出外层。
5. 遍历字符串 `"Hello, 世界"`，同时打印字节索引和字符。
6. 用无限循环和 `break` 实现一个猜数字游戏。

## 小结

- Go 只有 `for` 一种循环，条件循环、三段式和无限循环都用它表达。
- `range` 可以遍历整数（Go 1.22+）、字符串、数组、切片、map 和 channel。
- `break` 跳出当前循环，`continue` 跳过本次迭代。
- 嵌套循环可以用标签跳出多层。
- 循环条件必须是 `bool`，循环体必须使用花括号。

---

上一篇：[[Go/Basics/Constants|Go 基础：常量、无类型常量与 iota]]
下一篇：[[Go/Basics/If-Else|Go 基础：if/else 条件分支]]

返回 [[Go/Basics/index|Go 基础]]。
