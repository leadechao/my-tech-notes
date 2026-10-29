---
title: "Go 基础：常量、无类型常量与 iota"
description: "从 Go by Example 的 Constants 示例出发，理解 const、无类型常量的任意精度、显式转换、常量表达式与 iota 枚举。"
tags:
  - Go
  - 编程语言
  - 基础
  - 常量
created: 2026-10-29
modified: 2026-10-29
---

# Go 基础：常量、无类型常量与 iota

常量在编译期确定，运行期间不能被修改。Go 的常量设计有两个很有特点的地方：**无类型常量可以按上下文自动适配类型**，以及**常量表达式按高精度计算**。

[Go by Example 的 Constants](https://gobyexample.com/constants) 用一组数值演示了这两点。本文补充 typed/untyped 常量的区别、`iota` 枚举和常见错误。

## 完整示例

```go
package main

import (
	"fmt"
	"math"
)

const s string = "constant"

func main() {
	fmt.Println(s)

	const n = 500000000

	const d = 3e20 / n
	fmt.Println(d)

	fmt.Println(int64(d))

	fmt.Println(math.Sin(n))
}
```

运行：

```bash
$ go run constants.go
constant
6e+11
600000000000
-0.28470407323754404
```

## `const` 声明

常量用 `const` 声明，可以出现在包级别，也可以出现在函数内部：

```go
const s string = "constant" // 包级常量，显式指定 string

func main() {
	const n = 500000000 // 函数内常量，类型由字面量推断
	fmt.Println(n)
}
```

和 `var` 不同，`const` 声明的值必须在编译期就能确定：

- 可以是布尔、数值、字符串等基本类型的字面量；
- 可以是其他常量的运算结果；
- 不能是普通函数调用的结果；
- 不能是切片、map、结构体等复合值。

常量不能被重新赋值，也不能取地址：

```go
const s = "x"
s = "y"
// cannot assign to s (neither addressable nor a map index expression)

p := &s
// invalid operation: cannot take address of s
```

这两条限制来自同一个事实：常量不是运行时的内存对象，而是编译期的值。

## 无类型常量与高精度计算

`const n = 500000000` 和 `const d = 3e20 / n` 都没有写类型，它们属于**无类型常量**（untyped constant）。

无类型常量没有确定的 Go 类型，只有一个“种类”（整数、浮点数、字符串等）。在需要的时候，它会根据上下文转换成合适的类型；如果没有上下文，则使用默认类型。

常量表达式在语言层面按高精度计算。`3e20 / n` 的结果是精确的 `6e11`，不会在中间步骤损失精度：

```go
const n = 500000000
const d = 3e20 / n

fmt.Println(d)          // 6e+11
fmt.Println(int64(d))   // 600000000000
```

`fmt.Println` 接收的是 `any` 类型的参数，没有更具体的类型信息，因此 `d` 会使用默认类型 `float64`，输出为科学计数法 `6e+11`。如果想看到完整整数形式，可以格式化输出：

```go
fmt.Printf("%.0f\n", d) // 600000000000
```

`int64(d)` 则是在编译期把无类型常量转换成 `int64`，结果是精确的 `600000000000`。

## 默认类型与显式转换

无类型常量在脱离具体上下文时，会落到默认类型：

| 常量种类 | 默认类型 |
| --- | --- |
| 布尔 | `bool` |
| 字符（rune） | `rune`（`int32`） |
| 整数 | `int` |
| 浮点数 | `float64` |
| 复数 | `complex128` |
| 字符串 | `string` |

例如：

```go
const n = 500000000
fmt.Printf("%T\n", n) // int

const d = 3e20 / n
fmt.Printf("%T\n", d) // float64
```

但无类型常量的好处是：在需要其他类型时，只要值能够表示，就可以直接适配。

```go
const untyped = 1
var f float64 = untyped // 合法：1 可以表示为 float64
fmt.Println(f)          // 1
```

如果常量已经显式指定了类型，就不再具备这种灵活性：

```go
const typed int = 1
var g float64 = typed
// cannot use typed (constant 1 of type int) as float64 value
```

需要转换时，必须显式写出：

```go
var g float64 = float64(typed)
```

这也是 `math.Sin(n)` 能工作的原因：`math.Sin` 的参数类型是 `float64`，无类型整数常量 `n` 在调用处被转换成 `float64`。这不是一般意义上的隐式类型转换，而是常量在上下文中的适配。

## 常量表达式

常量表达式可以包含常见的算术、位运算和比较：

```go
const (
	KB = 1 << 10
	MB = 1 << 20
	GB = 1 << 30
)

const ok = 1 < 2
```

但普通函数调用不行：

```go
const x = math.Sin(1)
// math.Sin(1) (value of type float64) is not constant
```

因为 `math.Sin` 的结果要在运行时计算，编译器无法在编译期得到它。

一个很容易忽略的细节是：**整数常量之间的除法仍然是整数除法**。

```go
const ratio = 1 / 3   // 0
const ratiof = 1.0 / 3 // 0.3333333333333333
```

`1 / 3` 的两个操作数都是整数常量，结果先按整数除法得到 `0`，再赋给 `ratio`。想让常量表达式产生小数，至少要让其中一个操作数是浮点常量。

除以零的常量表达式会直接编译失败：

```go
const x = 1 / 0
// invalid operation: division by zero
```

## 类型化常量与溢出

常量可以显式指定类型：

```go
const Pi float64 = 3.14159
const MaxRetries int = 3
```

类型化常量的值必须能被目标类型表示，否则编译失败：

```go
const x int8 = 128
// cannot use 128 (untyped int constant) as int8 value in constant declaration (overflows)
```

无类型常量则可以暂时超出目标类型的范围，直到它被用在需要具体类型的地方：

```go
const big = 1 << 100 // 合法：无类型整数常量

fmt.Println(big)
// cannot use big (untyped int constant ...) as int value ... (overflows)
```

`big` 本身是一个合法的无类型常量，但 `fmt.Println` 需要把它落到默认类型 `int`，而 `int` 放不下 `1 << 100`，所以编译失败。

## 用 `iota` 生成枚举

`iota` 是常量声明中的计数器，每个 `const` 块从 `0` 开始，每遇到一个常量规格（通常是一行）递增一次，遇到新的 `const` 块则重置。

最常见的用法是定义枚举：

```go
type Weekday int

const (
	Sunday Weekday = iota
	Monday
	Tuesday
	Wednesday
	Thursday
	Friday
	Saturday
)
```

输出：

```go
fmt.Println(Sunday, Monday, Tuesday, Wednesday, Thursday, Friday, Saturday)
// 0 1 2 3 4 5 6
```

`Sunday` 显式写了 `iota`，后面的常量省略表达式，会复用上一行的表达式，因此依次得到 `1` 到 `6`。

`iota` 也可以参与位运算，用来定义标志位：

```go
const (
	Readable = 1 << iota // 1
	Writable             // 2
	Executable           // 4
)
```

用 `_` 可以跳过某个值：

```go
const (
	_  = iota             // 跳过 0
	KB = 1 << (10 * iota) // 1 << 10
	MB                    // 1 << 20
	GB                    // 1 << 30
)
```

`iota` 只在 `const` 声明中有意义，在普通变量声明里不可用。

> [!note]
> Go 的 `iota` 枚举只是“约定 + 自定义类型”，并不是 Java 那样的封闭枚举。`Weekday(99)` 这样的值在类型上仍然合法，需要额外的方法或校验来保证取值范围。

## 常量与变量如何选择

- 值在编译期就能确定，并且不会变化：用 `const`。
- 值依赖运行时输入、环境或计算结果：用 `var`。
- 需要表示一组相关的枚举值：用 `const` 块配合 `iota`。
- 需要切片、map、结构体等复合值：只能用 `var`，因为这些类型不能作为常量。

常量还有一个实际好处：它可以出现在数组长度、switch case 等要求编译期常量的位置。例如：

```go
const size = 10
var buf [size]byte
```

## 常见错误

### 1. `const ratio = 1 / 3` 得到 `0`

两个整数常量相除，结果仍是整数。使用 `1.0 / 3` 得到浮点结果。

### 2. 把类型化常量赋给另一种类型

`const typed int = 1` 不能直接赋给 `float64` 变量，需要显式转换。

### 3. 用函数调用初始化常量

`const x = math.Sin(1)` 不合法。常量必须在编译期可求值。

### 4. 修改常量或取地址

`const` 不是运行时对象，不能赋值，也不能用 `&` 取地址。

### 5. 用复合字面量声明常量

切片、map、结构体不能作为常量。需要这些类型时使用 `var`。

### 6. 类型化常量溢出

`const x int8 = 128` 会编译失败。选择足够宽的类型，或保持无类型常量到使用处再转换。

### 7. 误以为 `iota` 会跨 `const` 块继续递增

每个 `const` 块都会重新从 `0` 开始。

## 和 Java 对照

| 概念 | Go | Java |
| --- | --- | --- |
| 常量声明 | `const`，编译期求值 | `final`，可在运行期初始化 |
| 常量类型 | 类型化或无类型 | 每个变量都有确定类型 |
| 枚举 | `iota` + 自定义类型 | `enum` 类 |
| 不可变性 | 常量完全不可变 | `final` 引用不可变，对象内容可能可变 |
| 编译期常量 | 基本类型字面量和常量表达式 | 有限的编译期常量表达式 |

Java 的 `static final` 字段可以在运行时初始化，引用的对象内容也可能被修改；Go 的 `const` 则要求值在编译期就确定，且只支持基本类型。

## 动手练习

1. 声明一个包级字符串常量和一个函数内整数常量，分别打印。
2. 计算 `const d = 3e20 / 500000000`，分别用 `fmt.Println`、`int64(d)` 和 `%.0f` 输出。
3. 定义 `Weekday` 类型和七天的 `iota` 枚举，打印它们的值。
4. 用 `iota` 定义 `Readable`、`Writable`、`Executable` 三个位标志。
5. 对比 `const ratio = 1/3` 和 `const ratio = 1.0/3` 的结果。
6. 尝试把 `const typed int = 1` 赋给 `float64` 变量，观察编译错误，再用显式转换修复。

## 小结

- `const` 在编译期求值，不能被修改，也不能取地址。
- 无类型常量没有确定的 Go 类型，可以按上下文适配；脱离上下文时使用默认类型。
- 常量表达式按高精度计算，但整数常量除法仍然是整数除法。
- 类型化常量必须能被目标类型表示，否则编译失败。
- `iota` 用于生成递增的枚举值，每个 `const` 块重置。

---

上一篇：[[Go/Basics/Variables|Go 基础：变量声明与零值]]
下一篇：[[Go/Basics/For|Go 基础：for 循环与 range]]

返回 [[Go/Basics/index|Go 基础]]。
