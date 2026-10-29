---
title: "Go 基础：变量声明与零值"
description: "从 Go by Example 的 Variables 示例出发，掌握 var、短变量声明 :=、多变量赋值、零值、作用域与变量遮蔽。"
tags:
  - Go
  - 编程语言
  - 基础
  - 变量
created: 2026-10-29
modified: 2026-10-29
---

# Go 基础：变量声明与零值

在 Go 中，变量必须显式声明，编译器会据此检查类型是否正确。和某些脚本语言不同，`var a = "initial"` 并不会让 `a` 变成“任意类型”，它的类型在编译期就已经确定为 `string`。

[Go by Example 的 Variables](https://gobyexample.com/variables) 展示了 `var`、类型推断、零值和 `:=` 四种声明方式。本文补充作用域、变量遮蔽、多变量赋值和常见编译错误。

## 完整示例

```go
package main

import "fmt"

func main() {
	var a = "initial"
	fmt.Println(a)

	var b, c int = 1, 2
	fmt.Println(b, c)

	var d = true
	fmt.Println(d)

	var e int
	fmt.Println(e)

	f := "apple"
	fmt.Println(f)
}
```

运行：

```bash
$ go run variables.go
initial
1 2
true
0
apple
```

## 用 `var` 声明变量

最基本的声明形式是 `var`：

```go
var a = "initial"     // 类型推断为 string
var b, c int = 1, 2   // 显式指定 int，一次声明两个
var d = true          // 类型推断为 bool
var e int             // 只声明类型，使用零值
```

`var` 可以用在函数内部，也可以用在包级别。它既可以只声明类型，也可以同时初始化：

- 有初始值时，类型可以省略，由编译器推断；
- 没有初始值时，必须写明类型，变量会被赋予该类型的零值。

注意，`var e` 这种既不写类型也不写初始值的声明是不合法的，编译器无法知道 `e` 应该是什么类型。

## 类型推断不是动态类型

`var a = "initial"` 会让编译器推断出 `a` 是 `string`，而不是“运行时可变的任意类型”。之后不能把数字赋给它：

```go
var a = "initial"
a = 1
// cannot use 1 (untyped int constant) as string value in assignment
```

常见的字面量推断结果如下：

| 写法 | 推断类型 |
| --- | --- |
| `var a = "initial"` | `string` |
| `var b = 1` | `int` |
| `var c = 1.0` | `float64` |
| `var d = true` | `bool` |
| `var e = 'A'` | `rune`（`int32`） |

如果确实需要另一种类型，必须显式转换，例如 `var f float64 = float64(b)`。

## 零值

Go 不会留下“未初始化”的变量。声明时没有赋初值，编译器会自动填入该类型的零值：

| 类型 | 零值 |
| --- | --- |
| `int`、`int8`、`int16`、`int32`、`int64` | `0` |
| `uint`、`uint8`、`uint16`、`uint32`、`uint64` | `0` |
| `float32`、`float64` | `0` |
| `complex64`、`complex128` | `0 + 0i` |
| `bool` | `false` |
| `string` | `""` |
| 指针、切片、map、channel、函数、接口 | `nil` |
| 数组 | 每个元素都是对应类型的零值 |
| 结构体 | 每个字段都是对应类型的零值 |

因此：

```go
var e int
fmt.Println(e) // 0

var s string
fmt.Println(s == "") // true

var ok bool
fmt.Println(ok) // false
```

零值让变量在声明后就可以安全使用，但有一个重要例外：**nil map 可以读，不能写**。

```go
var m map[string]int
fmt.Println(m == nil, len(m)) // true 0

// m["go"] = 1
// panic: assignment to entry in nil map
```

需要先初始化：

```go
m := make(map[string]int)
m["go"] = 1
```

切片、channel 也有类似的 nil 语义，但 map 的写入 panic 最容易被忽略。结构体中的 `sync.Mutex` 等类型则可以直接使用零值。

## `:=` 短变量声明

在函数内部，可以用 `:=` 同时完成声明和初始化：

```go
f := "apple"
fmt.Println(f)
```

它等价于：

```go
var f string = "apple"
```

`:=` 有三个限制：

1. **只能用在函数内部**。包级别不能使用，否则会报 `non-declaration statement outside function body`。
2. **左边至少有一个新变量**。`a := 1; a := 2` 会报 `no new variables on left side of :=`。
3. **不能用来改变已有变量的类型**。`a := 1` 之后，`a` 已经是 `int`，不能再把字符串赋给它。

第二条限制有一个常见用法：只要左边有一个新变量，其他变量就可以复用：

```go
a, b := 1, 2
b, c := 3, 4
fmt.Println(a, b, c) // 1 3 4
```

这里 `c` 是新变量，所以 `b, c := 3, 4` 合法；`b` 被重新赋值，类型仍然是 `int`。

## 多变量赋值

Go 支持一次给多个变量赋值，右侧会先全部求值，再赋给左侧：

```go
a, b := 1, 2
a, b = b, a
fmt.Println(a, b) // 2 1
```

这个特性让交换变量不需要临时变量。

如果某个返回值不需要，可以用空白标识符 `_` 丢弃：

```go
_, err := fmt.Println("hello")
if err != nil {
	fmt.Println(err)
}
```

`fmt.Println` 返回写入的字节数和错误；这里只关心错误，所以用 `_` 忽略第一个返回值。

## 作用域与变量遮蔽

变量只在声明它的代码块内可见。函数内部用 `:=` 声明的变量，作用域从声明处开始，到所在代码块结束。

如果在内层代码块中声明了同名变量，就会发生**变量遮蔽**（shadowing）：

```go
x := 1
if true {
	x := 2
	fmt.Println(x) // 2
}
fmt.Println(x) // 1
```

内层的 `x` 是一个新变量，外层的 `x` 没有被修改。想修改外层变量，应使用 `=`：

```go
x := 1
if true {
	x = 2
}
fmt.Println(x) // 2
```

变量遮蔽在错误处理里尤其容易出问题。比如外层已经有一个 `err`，在 `if` 里写 `data, err := load()`，因为 `data` 是新变量，`:=` 仍然会创建一个新的 `err`，外层的 `err` 不会被赋值。需要复用外层变量时，先声明 `data`，再使用 `data, err = load()`。

> [!tip]
> 看到 `:=` 时，先确认左边是否真的有你以为的那个变量。编译器的类型检查能发现类型错误，但发现不了“遮蔽后逻辑不符合预期”的问题。

## 未使用变量

Go 对局部变量有严格限制：声明后没有使用，编译直接失败。

```go
func main() {
	var x int
}
// declared and not used: x
```

这里的“使用”包括读取变量、传给函数、参与运算等；只给它赋值不算使用：

```go
var x int
x = 1
// declared and not used: x
```

可以用 `_ = x` 明确表示丢弃，但更好的做法是删除不需要的变量。包级别的变量不受这条规则限制，未使用的包级变量不会导致编译失败。

## 包级变量

`var` 可以用在包级别，作用域是整个包。首字母大写的变量会被导出，其他包可以通过 `包名.变量名` 访问；首字母小写的变量只在包内可见。

```go
var (
	appName = "demo"
	version = "0.1.0"
)
```

包级变量可以跨文件使用。多个文件属于同一个包时，它们共享包级作用域。

## 常见错误

### 1. `var e` 没有类型也没有初值

声明变量时至少要提供类型或初始值，否则编译器无法推断。

### 2. 包级别使用 `:=`

`:=` 只能出现在函数内部。包级别应使用 `var`。

### 3. `no new variables on left side of :=`

同一作用域内重复使用 `:=`，且左边没有新变量。改用 `=` 赋值。

### 4. `declared and not used`

局部变量声明后未使用。删除变量，或用 `_ = x` 明确丢弃。

### 5. 向 nil map 写入

`var m map[string]int` 声明后 `m` 是 nil，读取安全，写入会 panic。使用 `make` 初始化。

### 6. 变量遮蔽导致赋值丢失

内层 `:=` 创建了新变量，外层变量没有被修改。检查是否需要改用 `=`。

## 和 Java 对照

| 概念 | Go | Java |
| --- | --- | --- |
| 局部变量声明 | `var x int` 或 `x := 1` | `int x = 1`，Java 10+ 可用 `var` |
| 未初始化 | 自动赋零值 | 局部变量必须先初始化；字段有默认值 |
| 类型推断 | `var` 和 `:=` | `var` 仅限局部变量 |
| 常量 | `const` | `final` |
| 可见性 | 首字母大小写 | `public`、`private` 等 |
| 未使用变量 | 局部变量编译错误 | 通常只是警告 |
| nil 引用 | nil map 写入会 panic | `null` 调用方法会抛 NPE |

Go 的零值机制减少了“未初始化”的不确定性，但也要求开发者理解 nil map、nil 切片等类型的边界。

## 动手练习

1. 分别用 `var` 和 `:=` 声明字符串、整数、浮点数和布尔变量，用 `%T` 打印类型。
2. 声明两个整数并交换它们的值，不使用临时变量。
3. 用 `var (...)` 声明一组包级变量，并区分导出和非导出变量。
4. 写一个内层代码块，用 `:=` 遮蔽外层变量，观察内外层输出。
5. 声明一个 nil map，先读取长度，再尝试写入，观察 panic。
6. 用 `_` 接收一个不需要的返回值。

## 小结

- `var` 可以声明一个或多个变量，支持类型推断和零值初始化。
- `:=` 是函数内部的声明加初始化简写，左边至少要有一个新变量。
- 未初始化的变量会得到零值；nil map 可读不可写。
- 变量有块级作用域，内层 `:=` 会造成变量遮蔽。
- Go 要求局部变量必须被使用，这能帮助清理死代码。

---

上一篇：[[Go/Basics/Values|Go 基础：值类型与字面量]]
下一篇：[[Go/Basics/Constants|Go 基础：常量、无类型常量与 iota]]

返回 [[Go/Basics/index|Go 基础]]。
