---
title: "Go 基础：值类型与字面量"
description: "从 Go by Example 的 Values 示例出发，理解字符串、整数、浮点数、布尔值、默认类型与显式类型转换。"
tags:
  - Go
  - 编程语言
  - 基础
  - 值类型
created: 2026-10-29
modified: 2026-10-29
---

# Go 基础：值类型与字面量

Go 是静态类型语言：变量一旦确定类型，就不能在运行过程中变成另一种类型。编译器会利用类型信息检查函数调用、运算符和赋值是否合法。

[Go by Example 的 Values](https://gobyexample.com/values) 用几行代码展示了最常用的值类型。本文在此基础上补充默认类型、整数除法、浮点精度和类型转换这几个新手最容易踩坑的地方。

## 完整示例

```go
package main

import "fmt"

func main() {
	fmt.Println("go" + "lang")
	fmt.Println("1+1 =", 1+1)
	fmt.Println("7.0/3.0 =", 7.0/3.0)
	fmt.Println(true && false)
	fmt.Println(true || false)
	fmt.Println(!true)
}
```

运行：

```bash
$ go run values.go
golang
1+1 = 2
7.0/3.0 = 2.3333333333333335
false
true
false
```

这几行分别覆盖了字符串、整数、浮点数和布尔值。下面逐个展开。

## 字符串

字符串用双引号表示，可以用 `+` 拼接：

```go
fmt.Println("go" + "lang") // golang
```

Go 的字符串是不可变的字节序列，通常存放 UTF-8 编码的文本。`len` 返回的是**字节数**，不是字符数：

```go
fmt.Println(len("go"))     // 2
fmt.Println(len("你好"))   // 6
```

一个中文字符在 UTF-8 中通常占 3 个字节，所以 `"你好"` 的长度是 6。如果需要按 Unicode 字符计数，应使用 `unicode/utf8` 包中的 `utf8.RuneCountInString`。

字符串只能和字符串相加，Go 不会自动把数字转成字符串：

```go
fmt.Println("1" + 1)
// invalid operation: "1" + 1 (mismatched types untyped string and untyped int)
```

如果需要把数字转成字符串，应使用 `strconv.Itoa` 或 `fmt.Sprint`。这个细节会在后面的“类型转换”一节继续展开。

## 整数

没有小数点的整数字面量默认是 `int` 类型：

```go
fmt.Println(1 + 1) // 2
```

`int` 的宽度与平台有关：在 64 位平台上通常是 64 位，在 32 位平台上则是 32 位。需要明确宽度时，可以使用 `int8`、`int16`、`int32`、`int64`，以及对应的无符号类型 `uint8`、`uint16`、`uint32`、`uint64`。

整数除法会向零截断，小数部分直接丢弃：

| 表达式 | 结果 |
| --- | --- |
| `7 / 3` | `2` |
| `-7 / 3` | `-2` |
| `7 % 3` | `1` |
| `-7 % 3` | `-1` |

因此 `1 / 2` 的结果是 `0`，不是 `0.5`。想要得到小数结果，至少要让其中一个操作数是浮点数：

```go
fmt.Println(1 / 2)     // 0
fmt.Println(1.0 / 2)   // 0.5
fmt.Println(1 / 2.0)   // 0.5
```

这是从其他语言迁移过来时非常常见的错误，尤其是计算比例、平均值的时候。

## 浮点数

带小数点的字面量默认是 `float64`：

```go
fmt.Println("7.0/3.0 =", 7.0/3.0)
// 7.0/3.0 = 2.3333333333333335
```

`float64` 使用二进制浮点表示，很多十进制小数无法被精确存储。例如：

```go
a, b := 0.1, 0.2
fmt.Println(a + b)          // 0.30000000000000004
fmt.Println(a+b == 0.3)     // false
```

注意这里的 `a` 和 `b` 是变量，运算发生在运行时，结果遵循 `float64` 的精度。如果直接写常量表达式 `0.1 + 0.2`，Go 会在编译期以任意精度计算，再转换到目标类型，结果可能看起来“正确”，但不要依赖这种差异。

比较浮点数时，通常不直接用 `==`，而是判断差值是否足够小：

```go
if math.Abs(a+b-0.3) < 1e-9 {
	fmt.Println("近似相等")
}
```

格式化输出可以控制小数位数：

```go
fmt.Printf("%.2f\n", 7.0/3.0) // 2.33
```

`float32` 的精度更低，除非有明确的存储或性能需求，否则默认使用 `float64`。

## 布尔值

布尔类型只有两个值：`true` 和 `false`。常用运算符有：

- `&&`：逻辑与，两边都为真时为真；
- `||`：逻辑或，任意一边为真时为真；
- `!`：逻辑非。

```go
fmt.Println(true && false) // false
fmt.Println(true || false) // true
fmt.Println(!true)         // false
```

`&&` 和 `||` 具有短路行为：如果左边的结果已经能决定最终结果，右边就不会执行。例如 `false && f()` 不会调用 `f()`。

Go 没有“真值”概念，条件必须是真正的 `bool`：

```go
n := 1
if n {
}
// non-boolean condition in if statement
```

应该写成 `if n != 0`。这一点和 Java 一致，但和 JavaScript、Python 不同。

## 字面量的默认类型

当字面量需要落到一个具体类型时，Go 会使用它的默认类型：

| 字面量 | 默认类型 |
| --- | --- |
| `"golang"` | `string` |
| `1` | `int` |
| `1.0` | `float64` |
| `true` | `bool` |
| `'A'` | `rune`（即 `int32`） |

可以用 `%T` 查看变量的实际类型：

```go
fmt.Printf("%T %T %T %T\n", "golang", 1, 7.0/3.0, true)
// string int float64 bool
```

单引号包裹的是字符字面量，类型是 `rune`，表示一个 Unicode 码点；双引号包裹的才是字符串。

## 显式类型转换

Go 不允许在两种已确定类型的数值之间隐式转换：

```go
var i int = 1
var f float64 = 1.0

fmt.Println(i + f)
// invalid operation: i + f (mismatched types int and float64)

fmt.Println(float64(i) + f) // 2
```

必须显式写出目标类型。这个规则同样适用于函数参数和返回值，编译器不会替你做“看起来合理”的转换。

浮点数转整数会丢弃小数部分，而不是四舍五入：

```go
f := 3.9
g := -3.9
fmt.Println(int(f)) // 3
fmt.Println(int(g)) // -3
```

字符串和整数之间的转换尤其容易误解：

```go
fmt.Println(string(65))       // A
fmt.Println(strconv.Itoa(65)) // 65
```

`string(65)` 把整数当作 Unicode 码点转换成字符，而不是转成十进制文本。要把整数转成数字字符串，应使用 `strconv.Itoa`；反过来，把字符串转成整数使用 `strconv.Atoi`。

> [!warning]
> 无类型常量是一个例外：`var i int = 1; i + 1.0` 是合法的，因为 `1.0` 是无类型常量，可以按上下文转换成 `int`。但如果 `1.5` 无法表示为整数，就会编译失败。不要把这种常量转换理解成一般的隐式类型转换。

## 常见错误

### 1. `1 + "1"`

字符串和数字不能相加。需要先把数字转成字符串，或把字符串解析成数字。

### 2. `1 / 2` 得到 `0`

两个整数相除，结果仍是整数。想得到小数，至少使用一个浮点数。

### 3. `if 1`

条件必须是 `bool`。整数、字符串、指针都不会自动转换成布尔值。

### 4. `string(65)` 得到 `"A"`

`string(int)` 是码点转换，不是十进制格式化。使用 `strconv.Itoa`。

### 5. 用 `==` 比较浮点计算结果

二进制浮点表示存在误差，应使用误差范围比较。

## 和 Java 对照

| 概念 | Go | Java |
| --- | --- | --- |
| 整数默认类型 | `int`（平台相关，32 或 64 位） | `int`（固定 32 位） |
| 浮点默认类型 | `float64` | `double` |
| 字符类型 | `rune`（`int32`，Unicode 码点） | `char`（16 位 UTF-16 单元） |
| 字符串编码 | UTF-8 字节序列 | UTF-16 |
| 数值转换 | 必须显式转换 | 允许部分拓宽转换 |
| 条件表达式 | 只能是 `bool` | 只能是 `boolean` |
| 整数除法 | 向零截断 | 向零截断 |

Java 允许 `int` 自动拓宽为 `long`、`double`，Go 不提供这种隐式转换；写 Go 时需要把类型转换显式表达出来。

## 动手练习

1. 用 `%T` 打印 `"golang"`、`1`、`1.0`、`true`、`'A'` 的类型。
2. 分别计算 `1/2`、`1.0/2`、`1/2.0`，解释结果差异。
3. 用 `strconv.Itoa` 把整数 `65` 转成字符串，再和 `string(65)` 的结果对比。
4. 声明两个 `float64` 变量保存 `0.1` 和 `0.2`，比较它们的和与 `0.3`，再用误差范围判断。
5. 写一个返回 `bool` 的函数，验证 `false && f()` 不会调用 `f()`。

## 小结

- Go 的常用值类型包括字符串、整数、浮点数和布尔值。
- 字面量有默认类型：`1` 是 `int`，`1.0` 是 `float64`，`'A'` 是 `rune`。
- 整数除法会截断，浮点数存在精度误差。
- Go 的数值转换必须显式进行，`string(int)` 和 `strconv.Itoa` 的含义完全不同。
- 布尔条件只能是 `bool`，不存在“真值”转换。

---

上一篇：[[Go/Basics/Hello World|Go 入门：从 Hello World 开始]]
下一篇：[[Go/Basics/Variables|Go 基础：变量声明与零值]]

返回 [[Go/Basics/index|Go 基础]]。
