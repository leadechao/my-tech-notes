---
title: "Go 基础：switch 与类型分支"
description: "从 Go by Example 的 Switch 示例出发，掌握表达式 switch、多值 case、无表达式 switch、fallthrough 和类型 switch。"
tags:
  - Go
  - 编程语言
  - 基础
  - 流程控制
created: 2026-10-29
modified: 2026-10-29
---

# Go 基础：switch 与类型分支

分支一多，`if/else if` 就会变得冗长。`switch` 把多个分支集中在一个结构里，既清晰又不容易漏掉 `else`。

Go 的 `switch` 比 Java 更灵活：默认不贯穿、支持多值 case、可以省略判断表达式，还能做类型分支。

[Go by Example 的 Switch](https://gobyexample.com/switch) 覆盖了这些用法。本文补充 `fallthrough`、初始化语句、类型 switch 的细节和常见错误。

## 完整示例

```go
package main

import (
	"fmt"
	"time"
)

func main() {
	i := 2
	fmt.Print("Write ", i, " as ")
	switch i {
	case 1:
		fmt.Println("one")
	case 2:
		fmt.Println("two")
	case 3:
		fmt.Println("three")
	}

	switch time.Now().Weekday() {
	case time.Saturday, time.Sunday:
		fmt.Println("It's the weekend")
	default:
		fmt.Println("It's a weekday")
	}

	t := time.Now()
	switch {
	case t.Hour() < 12:
		fmt.Println("It's before noon")
	default:
		fmt.Println("It's after noon")
	}

	whatAmI := func(i any) {
		switch t := i.(type) {
		case bool:
			fmt.Println("I'm a bool")
		case int:
			fmt.Println("I'm an int")
		default:
			fmt.Printf("Don't know type %T\n", t)
		}
	}
	whatAmI(true)
	whatAmI(1)
	whatAmI("hey")
}
```

某次运行的输出（周末/工作日、上午/下午取决于运行时间）：

```bash
$ go run switch.go
Write 2 as two
It's a weekday
It's after noon
I'm a bool
I'm an int
Don't know type string
```

## 基本形式

```go
switch i {
case 1:
	fmt.Println("one")
case 2:
	fmt.Println("two")
case 3:
	fmt.Println("three")
}
```

`switch` 后面的表达式可以是变量、函数调用或字面量。每个 `case` 的值与它比较，匹配后执行对应分支，然后自动结束整个 `switch`。

这里最关键的区别是：**Go 的 case 默认不会贯穿**。执行完 `case 2` 后，不会继续执行 `case 3`。不需要像 C、Java 那样在末尾写 `break`。

`default` 是可选的，表示所有 case 都不匹配时执行：

```go
switch n {
case 0:
	fmt.Println("zero")
default:
	fmt.Println("non-zero")
}
```

`default` 不一定写在最后，但一个 `switch` 只能有一个 `default`。

## 多值 case

一个 case 可以匹配多个值，用逗号分隔：

```go
switch time.Now().Weekday() {
case time.Saturday, time.Sunday:
	fmt.Println("It's the weekend")
default:
	fmt.Println("It's a weekday")
}
```

这比写两个重复的 case 分支更简洁。case 后面的表达式不必是常量，也可以是函数调用或其他表达式；不过重复的常量 case 会编译失败。

## 无表达式 switch

`switch` 后面可以什么都不写：

```go
t := time.Now()
switch {
case t.Hour() < 12:
	fmt.Println("It's before noon")
default:
	fmt.Println("It's after noon")
}
```

这种写法等价于 `switch true`，每个 case 都是一个布尔表达式。它适合替代很长的 `if/else if` 链，而且没有 `fallthrough` 的干扰。

case 表达式必须是 `bool`：

```go
switch {
case 1:
}
// cannot convert 1 (untyped int constant) to type bool
```

## 带初始化语句的 switch

和 `if` 一样，`switch` 可以带一条初始化语句：

```go
switch x := compute(); x {
case 0:
	fmt.Println("zero")
case 1:
	fmt.Println("one")
default:
	fmt.Println("many")
}
```

`x` 的作用域覆盖整个 `switch`，包括所有 case 和 default，但在 `switch` 之后不可见。

## `fallthrough`：显式贯穿

如果确实需要继续执行下一个 case，可以使用 `fallthrough`：

```go
switch 1 {
case 1:
	fmt.Println("one")
	fallthrough
case 2:
	fmt.Println("two")
	fallthrough
case 3:
	fmt.Println("three")
}
```

输出：

```text
one
two
three
```

`fallthrough` 会无条件进入下一个 case，不再判断下一个 case 的条件。它有几个限制：

- 必须是 case 分支的最后一条语句；
- 不能用在最后一个 case；
- 不能用在类型 switch 中。

大多数情况下不需要 `fallthrough`。如果发现自己在用它，先确认是否真的需要贯穿，还是分支逻辑可以重新组织。

## 类型 switch

类型 switch 用来判断接口值的动态类型：

```go
whatAmI := func(i any) {
	switch t := i.(type) {
	case bool:
		fmt.Println("I'm a bool")
	case int:
		fmt.Println("I'm an int")
	default:
		fmt.Printf("Don't know type %T\n", t)
	}
}
```

`i.(type)` 只能出现在 `switch` 中。每个 case 里的 `t` 会被赋予对应的具体类型：

- `case bool` 中，`t` 是 `bool`；
- `case int` 中，`t` 是 `int`；
- `default` 中，`t` 仍然是接口类型 `any`，但 `%T` 可以打印它的动态类型。

类型 switch 也可以匹配 `nil`：

```go
switch v := x.(type) {
case nil:
	fmt.Println("nil interface")
default:
	fmt.Printf("%T\n", v)
}
```

类型 switch 不能使用 `fallthrough`，因为不同分支的变量类型不同，贯穿没有意义。

## `break` 在 switch 中的行为

Go 的 case 默认不贯穿，所以通常不需要 `break`。但在 case 中提前结束 switch 仍然可以使用 `break`：

```go
switch n {
case 1:
	if done {
		break
	}
	fmt.Println("continue handling")
}
```

如果在循环里写 `switch`，`break` 只会跳出 `switch`，不会跳出外层循环。要跳出循环，需要使用标签：

```go
loop:
for _, n := range nums {
	switch n {
	case 0:
		break loop
	}
}
```

## 常见错误

### 1. 以为 case 会贯穿

Go 的 case 执行完自动结束。不要照搬 C/Java 的习惯在末尾写 `break`，也不要期待不写 `fallthrough` 会继续执行下一个 case。

### 2. 在最后一个 case 使用 `fallthrough`

```go
switch 1 {
case 1:
	fallthrough
}
// cannot fallthrough final case in switch
```

### 3. 在类型 switch 中使用 `fallthrough`

类型 switch 的每个分支类型不同，不允许贯穿。

### 4. case 类型不匹配

`switch` 表达式的类型和 case 值的类型必须可比较。比如 `switch 1 { case "1": }` 会编译失败。

### 5. 重复的常量 case

```go
switch 1 {
case 1:
case 1:
}
// duplicate case 1 (constant of type int) in expression switch
```

### 6. 无表达式 switch 的 case 不是布尔值

`switch { case 1: }` 会报 `cannot convert 1 ... to type bool`。无表达式 switch 的 case 必须是布尔表达式。

### 7. 以为 `break` 会跳出外层循环

`break` 只退出当前 `switch`。嵌套在循环里时，需要标签才能跳出循环。

## 和 Java 对照

| 概念 | Go | Java |
| --- | --- | --- |
| 默认贯穿 | 不贯穿 | 贯穿，需要 `break` |
| 多值 case | `case a, b:` | 传统 switch 不支持，需多个 case |
| case 表达式 | 可以是表达式 | 通常是编译期常量 |
| 无表达式 switch | 支持 | 不支持 |
| 类型 switch | 支持 | 无直接等价语法 |
| 可切换类型 | 任意可比较类型 | 受版本限制（传统为整数、枚举、字符串等） |
| `fallthrough` | 显式关键字 | 默认行为 |

Java 的 switch 需要 `break` 来避免贯穿，Go 则相反：默认不贯穿，需要 `fallthrough` 才贯穿。这个差异在迁移代码时最容易出错。

## 动手练习

1. 用 `switch` 把数字 `1` 到 `5` 转成英文单词。
2. 用 `time.Now().Weekday()` 判断周末或工作日。
3. 用无表达式 `switch` 实现成绩等级。
4. 写一个带 `fallthrough` 的 switch，先预测输出，再运行验证。
5. 用类型 switch 判断 `int`、`string`、`bool` 和其他类型。
6. 在循环中的 switch 里使用标签 `break`，观察它如何跳出外层循环。

## 小结

- Go 的 `switch` 默认不贯穿，执行完匹配的 case 自动结束。
- 一个 case 可以匹配多个值；`default` 可选。
- 无表达式 `switch` 等价于 `switch true`，适合替代长 `if/else` 链。
- `fallthrough` 显式进入下一个 case，限制较多，通常不需要。
- 类型 switch 用于判断接口值的动态类型，每个 case 中的变量具有对应类型。

---

上一篇：[[Go/Basics/If-Else|Go 基础：if/else 条件分支]]
下一篇：[[Go/Basics/Arrays|Go 基础：数组]]

返回 [[Go/Basics/index|Go 基础]]。
