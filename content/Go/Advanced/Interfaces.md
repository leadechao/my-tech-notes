---
title: "Go 进阶：接口"
description: "从 Go by Example 的 Interfaces 示例出发，理解接口定义、隐式实现、多态、类型断言、空接口与 typed nil 陷阱。"
tags:
  - Go
  - 编程语言
  - 进阶
  - 接口
created: 2026-10-29
modified: 2026-10-29
---

# Go 进阶：接口

接口是一组方法签名的集合。它描述“能做什么”，而不关心“是什么类型”。

Go 的接口是隐式实现的：只要一个类型拥有接口要求的所有方法，它就自动满足该接口，不需要写 `implements`。这让类型之间的耦合更低，也让标准库的小接口非常容易复用。

[Go by Example 的 Interfaces](https://gobyexample.com/interfaces) 用几何图形演示了多态和类型断言。本文补充接口值、typed nil、空接口和设计建议。

## 完整示例

```go
package main

import (
	"fmt"
	"math"
)

type geometry interface {
	area() float64
	perim() float64
}

type rect struct {
	width, height float64
}

type circle struct {
	radius float64
}

func (r rect) area() float64 {
	return r.width * r.height
}

func (r rect) perim() float64 {
	return 2*r.width + 2*r.height
}

func (c circle) area() float64 {
	return math.Pi * c.radius * c.radius
}

func (c circle) perim() float64 {
	return 2 * math.Pi * c.radius
}

func measure(g geometry) {
	fmt.Println(g)
	fmt.Println(g.area())
	fmt.Println(g.perim())
}

func detectCircle(g geometry) {
	if c, ok := g.(circle); ok {
		fmt.Println("circle with radius", c.radius)
	}
}

func main() {
	r := rect{width: 3, height: 4}
	c := circle{radius: 5}

	measure(r)
	measure(c)

	detectCircle(r)
	detectCircle(c)
}
```

运行：

```bash
$ go run interfaces.go
{3 4}
12
14
{5}
78.53981633974483
31.41592653589793
circle with radius 5
```

## 定义接口

```go
type geometry interface {
	area() float64
	perim() float64
}
```

接口里只写方法签名，不写实现。任何类型，只要实现了 `area()` 和 `perim()` 两个方法，就满足 `geometry`。

接口可以嵌入其他接口：

```go
type reader interface {
	read() string
}

type writer interface {
	write(s string)
}

type readWriter interface {
	reader
	writer
}
```

`readWriter` 要求同时实现 `read` 和 `write`。

## 隐式实现

`rect` 和 `circle` 都没有声明“实现 geometry”，但因为它们定义了同名、同签名的方法，就自动满足接口：

```go
func (r rect) area() float64 { ... }
func (r rect) perim() float64 { ... }
```

这种隐式实现的好处是：接口可以由使用方定义，而不需要修改被使用的类型。标准库的 `io.Reader`、`fmt.Stringer` 都是这样工作的。

## 用接口实现多态

```go
func measure(g geometry) {
	fmt.Println(g.area())
	fmt.Println(g.perim())
}
```

`measure` 不关心传入的是 `rect` 还是 `circle`，只要求它满足 `geometry`。调用时：

```go
measure(r)
measure(c)
```

同一个函数对不同类型执行不同的方法实现，这就是多态。

## 接口值

接口变量包含两部分信息：

- 动态类型：当前存储的具体类型；
- 动态值：该类型的一个值。

```go
var g geometry = rect{width: 3, height: 4}
fmt.Printf("%T\n", g) // main.rect
```

接口变量可以重新赋值为另一个实现了该接口的类型：

```go
g = circle{radius: 5}
fmt.Printf("%T\n", g) // main.circle
```

调用 `g.area()` 时，运行时会根据动态类型找到对应的方法。

## 类型断言

有时需要从接口值中取出具体类型。使用类型断言：

```go
c, ok := g.(circle)
if ok {
	fmt.Println("circle with radius", c.radius)
}
```

带 `ok` 的形式是安全的：断言失败时 `ok` 为 `false`，`c` 是零值，不会 panic。

不带 `ok` 的形式在失败时会 panic：

```go
c := g.(circle)
// 如果 g 的动态类型不是 circle，会 panic
```

只有在确定类型时才使用不带 `ok` 的断言。

类型断言也可以断言到另一个接口：

```go
if s, ok := g.(fmt.Stringer); ok {
	fmt.Println(s.String())
}
```

## 类型 switch

需要判断多种类型时，使用类型 switch：

```go
switch v := g.(type) {
case rect:
	fmt.Println("rect", v.width, v.height)
case circle:
	fmt.Println("circle", v.radius)
default:
	fmt.Printf("unknown %T\n", v)
}
```

类型 switch 的详细用法见 switch 一篇。

## 空接口与 `any`

空接口没有任何方法要求，因此所有类型都满足它：

```go
var x interface{}
x = 1
x = "hello"
x = rect{}
```

Go 1.18 起，`any` 是 `interface{}` 的别名：

```go
func printAny(v any) {
	fmt.Println(v)
}
```

`any` 适合确实需要接受任意类型的场景，例如 `fmt.Println`、JSON 解码。但滥用 `any` 会失去类型检查，调用方需要频繁断言，通常应优先使用具体接口或泛型。

## typed nil 陷阱

接口值为 nil 的条件是：动态类型和动态值都为 nil。一个指向 nil 的指针赋给接口后，接口本身并不等于 nil：

```go
type MyType struct{}

var p *MyType
var i any = p

fmt.Println(p == nil) // true
fmt.Println(i == nil) // false
```

因为 `i` 的动态类型是 `*MyType`，动态值是 nil。此时调用接口方法可能触发 panic。

这类问题常见于函数返回 `error`：

```go
func do() error {
	var p *MyError
	return p // 返回的 error 不是 nil
}
```

如果 `*MyError` 实现了 `error`，调用方检查 `err != nil` 会得到 true，即使底层指针是 nil。正确做法是直接返回 `nil`，或确保不把 nil 指针装入接口。

## 接口比较

接口值可以用 `==` 比较，但要求动态类型可比较。如果动态类型是切片、map 等不可比较类型，比较会 panic：

```go
var a any = []int{1}
var b any = []int{1}
fmt.Println(a == b)
// panic: comparing uncomparable type []int
```

比较接口前要确认动态类型可比较，或者使用 `reflect.DeepEqual`（仅适合测试和调试）。

## 小接口与设计建议

Go 提倡小而专注的接口：

```go
type Reader interface {
	Read(p []byte) (n int, err error)
}

type Writer interface {
	Write(p []byte) (n int, err error)
}
```

小接口更容易被实现和组合。常见建议：

- 在使用方定义接口，而不是实现方；
- 接口尽量小，一个或两个方法即可；
- 返回具体类型，接受接口；
- 不要为了“以后可能扩展”而提前定义大接口。

## 常见错误

### 1. 指针接收者导致类型不满足接口

如果接口方法由 `*T` 实现，只有 `*T` 满足接口，`T` 不满足。检查方法集。

### 2. 类型断言失败 panic

不确定类型时使用 `v, ok := i.(T)`，不要直接 `i.(T)`。

### 3. 把 nil 指针赋给接口

接口不为 nil，但内部指针是 nil。返回 `error` 时要特别小心。

### 4. 滥用 `any`

`any` 会丢失类型信息，导致大量断言和运行时错误。能用具体接口或泛型时优先使用它们。

### 5. 比较包含不可比较动态值的接口

接口比较可能 panic。只比较可比较的动态类型。

### 6. 接口定义过大

大接口难以实现和 mocking。拆成小接口，按需组合。

## 和 Java 对照

| 概念 | Go | Java |
| --- | --- | --- |
| 实现方式 | 隐式，满足方法集即可 | 显式 `implements` |
| 继承 | 不支持，接口可嵌入 | 接口可继承 |
| 多态 | 接口值 + 动态派发 | 接口/父类引用 + 动态派发 |
| 空类型 | `any` / `interface{}` | `Object` |
| 类型判断 | 类型断言 / 类型 switch | `instanceof` / 模式匹配 |
| 默认方法 | 不支持 | 接口可有 `default` 方法 |
| nil | nil 接口 / typed nil | null |

Java 的接口实现是显式声明，Go 是隐式的；Java 的 `instanceof` 对应 Go 的类型断言，Java 的 `Object` 对应 `any`。Java 接口可以有默认方法，Go 接口只有方法签名。

## 动手练习

1. 定义 `Shape` 接口，包含 `Area()` 和 `Perimeter()`，让矩形和圆形实现它。
2. 写一个接收 `Shape` 的函数，分别传入两种图形。
3. 用类型断言判断接口值是否为圆形。
4. 用类型 switch 处理 `int`、`string` 和自定义类型。
5. 构造一个 typed nil 例子，验证接口不等于 nil。
6. 把一个大接口拆成两个小接口，并让一个类型同时实现它们。

## 小结

- 接口是方法签名的集合，描述行为而不是数据。
- Go 的接口是隐式实现的，只要方法集满足即可。
- 接口值包含动态类型和动态值，支持多态调用。
- 类型断言和类型 switch 用于取出具体类型。
- `any` 是所有类型都满足的空接口，但不应滥用。
- nil 指针装入接口后，接口本身不等于 nil。
- 小接口更容易实现、组合和测试。

---

上一篇：[[Go/Advanced/Methods|Go 进阶：方法]]
下一篇：[[Go/Advanced/Enums|Go 进阶：枚举]]

返回 [[Go/Advanced/index|Go 进阶]]。
