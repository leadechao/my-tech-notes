---
title: "Go 基础：if/else 条件分支"
description: "从 Go by Example 的 If/Else 示例出发，理解条件写法、初始化语句、分支作用域、else 的位置要求，以及为什么 Go 没有三元运算符。"
tags:
  - Go
  - 编程语言
  - 基础
  - 流程控制
created: 2026-10-29
modified: 2026-10-29
---

# Go 基础：if/else 条件分支

`if` 是最基本的控制流语句。Go 的 `if` 语法看起来和 Java、C 很像，但有几个必须记住的差异：条件不需要括号、花括号不能省略、`else` 必须和前面的 `}` 在同一行，而且没有三元运算符。

[Go by Example 的 If/Else](https://gobyexample.com/if-else) 展示了基本分支、逻辑运算和带初始化语句的写法。本文补充作用域、短路求值和常见语法错误。

## 完整示例

```go
package main

import "fmt"

func main() {
	if 7%2 == 0 {
		fmt.Println("7 is even")
	} else {
		fmt.Println("7 is odd")
	}

	if 8%4 == 0 {
		fmt.Println("8 is divisible by 4")
	}

	if 8%2 == 0 || 7%2 == 0 {
		fmt.Println("either 8 or 7 are even")
	}

	if num := 9; num < 0 {
		fmt.Println(num, "is negative")
	} else if num < 10 {
		fmt.Println(num, "has 1 digit")
	} else {
		fmt.Println(num, "has multiple digits")
	}
}
```

运行：

```bash
$ go run if-else.go
7 is odd
8 is divisible by 4
either 8 or 7 are even
9 has 1 digit
```

## 基本写法

```go
if 7%2 == 0 {
	fmt.Println("7 is even")
} else {
	fmt.Println("7 is odd")
}
```

条件必须是 `bool` 表达式。和 Java 一样，整数、字符串、指针不会自动转换成布尔值；和 Java 不同的是，条件外面不需要括号。

`if` 可以没有 `else`：

```go
if 8%4 == 0 {
	fmt.Println("8 is divisible by 4")
}
```

即使分支里只有一条语句，花括号也不能省略。Go 没有“单条语句可以省略花括号”的规则。

## `else` 必须紧跟 `}`

下面这种写法在 Java 中很常见，但在 Go 中会编译失败：

```go
if ok {
	fmt.Println("ok")
}
else {
	fmt.Println("not ok")
}
```

报错：

```text
syntax error: unexpected keyword else, expected }
```

原因是 Go 会在 `}` 后自动插入分号，`else` 就被分号隔开了。必须写成：

```go
if ok {
	fmt.Println("ok")
} else {
	fmt.Println("not ok")
}
```

这是 Go 代码格式里少数不能靠个人风格决定的地方，`gofmt` 也会强制保持这种写法。

## 逻辑运算符与短路

条件中常用的逻辑运算符：

- `&&`：逻辑与；
- `||`：逻辑或；
- `!`：逻辑非。

```go
if 8%2 == 0 || 7%2 == 0 {
	fmt.Println("either 8 or 7 are even")
}
```

`&&` 和 `||` 都是短路的。左边已经能决定结果时，右边不会执行。这个特性常用于空指针保护：

```go
if user != nil && user.Name != "" {
	fmt.Println("hello", user.Name)
}
```

如果 `user` 是 nil，`user.Name` 不会被求值，因此不会触发 panic。把两个条件顺序写反，就可能在 nil 上取字段。

## 带初始化语句的 `if`

`if` 可以在条件前写一条初始化语句，用分号隔开：

```go
if num := 9; num < 0 {
	fmt.Println(num, "is negative")
} else if num < 10 {
	fmt.Println(num, "has 1 digit")
} else {
	fmt.Println(num, "has multiple digits")
}
```

`num` 的作用域包括整个 `if`、`else if` 和 `else` 块，但不包括 `if` 语句之后：

```go
if num := 9; num < 10 {
	fmt.Println(num)
}
// fmt.Println(num) // undefined: num
```

初始化语句不限于短变量声明，也可以是赋值或函数调用：

```go
if err := run(); err != nil {
	fmt.Println("failed:", err)
}
```

这种写法把“获取值”和“判断值”放在一起，能减少变量泄漏到外层作用域。错误处理里非常常见。

## 没有三元运算符

Go 没有 `condition ? a : b` 这样的三元表达式。最简单的替代是完整的 `if/else`：

```go
status := "unknown"
if ok {
	status = "ok"
} else {
	status = "failed"
}
```

如果只是二选一，也可以封装成小函数：

```go
func pick(ok bool, a, b string) string {
	if ok {
		return a
	}
	return b
}
```

Go 的设计者认为三元表达式会降低可读性，因此没有加入这个语法。遇到需要三元的地方，老老实实写 `if` 即可。

## 嵌套与 `else if`

多个条件可以用 `else if` 串起来：

```go
score := 85
if score >= 90 {
	fmt.Println("A")
} else if score >= 80 {
	fmt.Println("B")
} else if score >= 60 {
	fmt.Println("C")
} else {
	fmt.Println("D")
}
```

Go 没有 `switch` 的贯穿问题，分支很多时也可以改用 `switch`，可读性通常更好。

嵌套 `if` 时要注意作用域和变量遮蔽。内层用 `:=` 声明的同名变量会遮蔽外层变量：

```go
x := 1
if true {
	x := 2
	fmt.Println(x) // 2
}
fmt.Println(x) // 1
```

想修改外层变量，应使用 `=` 而不是 `:=`。

## 常见错误

### 1. `else` 写在下一行

Go 会自动插入分号，导致 `else` 无法与前面的 `if` 配对。必须写成 `} else {`。

### 2. 条件使用赋值 `=`

```go
if x = 1 {
}
// syntax error: cannot use assignment x = 1 as value
```

条件应该使用比较运算符 `==`。

### 3. 条件不是 `bool`

```go
if 1 {
}
// non-boolean condition in if statement
```

Go 没有真值转换，必须写成 `if n != 0`。

### 4. 省略花括号

```go
if ok
	fmt.Println("ok")
```

Go 要求花括号，即使只有一条语句。

### 5. 在 `if` 外使用初始化变量

`if num := 9; ...` 中的 `num` 只在该 `if` 语句内可见。需要在外面使用时，先声明变量：

```go
num := 9
if num < 10 {
	fmt.Println(num)
}
```

## 和 Java 对照

| 概念 | Go | Java |
| --- | --- | --- |
| 条件括号 | 不需要 | 必须 |
| 花括号 | 必须 | 单条语句可省略 |
| `else` 位置 | 必须紧跟 `}` | 可以换行 |
| 三元运算符 | 没有 | `condition ? a : b` |
| 条件类型 | `bool` | `boolean` |
| 初始化语句 | 支持 `if x := f(); ...` | 不支持 |
| 短路求值 | 支持 | 支持 |

Java 的 `if` 条件必须用括号包住，Go 则相反；Java 允许省略单条语句的花括号，Go 不允许。写 Go 时不要沿用 Java 的换行习惯，否则很容易在 `else` 上踩坑。

## 动手练习

1. 判断一个整数是奇数还是偶数。
2. 用 `else if` 实现百分制成绩等级。
3. 用 `if num := 9; ...` 判断数字位数，并尝试在 `if` 外打印 `num`，观察编译错误。
4. 写一个可能返回 nil 的指针，用 `&&` 做空指针保护。
5. 把一段 Java 风格的三元表达式改写成 Go 的 `if/else`。
6. 把多分支 `if/else` 改写成 `switch`，比较可读性。

## 小结

- `if` 条件不需要括号，但花括号必须存在。
- `else` 必须和前面的 `}` 在同一行。
- `if` 可以带初始化语句，变量作用域覆盖所有分支。
- Go 没有三元运算符，用完整的 `if/else` 替代。
- `&&` 和 `||` 具有短路行为，可用于 nil 保护。

---

上一篇：[[Go/Basics/For|Go 基础：for 循环与 range]]
下一篇：[[Go/Basics/Switch|Go 基础：switch 与类型分支]]

返回 [[Go/Basics/index|Go 基础]]。
