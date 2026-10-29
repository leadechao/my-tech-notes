---
title: "Go 进阶：指针"
description: "从 Go by Example 的 Pointers 示例出发，理解 &、*、指针参数、nil 指针、new、结构体指针与 Go 指针的使用边界。"
tags:
  - Go
  - 编程语言
  - 进阶
  - 指针
created: 2026-10-29
modified: 2026-10-29
---

# Go 进阶：指针

指针保存的是变量的内存地址。Go 的指针比 C 简单：没有指针运算，不能随意转换类型，垃圾回收器会管理对象生命周期。

指针最重要的用途是**让函数修改调用方的数据**，以及**避免复制大对象**。

[Go by Example 的 Pointers](https://gobyexample.com/pointers) 用两个置零函数对比了值传递和指针传递。本文补充 nil 指针、`new`、结构体指针和常见误区。

## 完整示例

```go
package main

import "fmt"

func zeroval(ival int) {
	ival = 0
}

func zeroptr(iptr *int) {
	*iptr = 0
}

func main() {
	i := 1
	fmt.Println("initial:", i)

	zeroval(i)
	fmt.Println("zeroval:", i)

	zeroptr(&i)
	fmt.Println("zeroptr:", i)

	fmt.Println("pointer:", &i)

	p := new(int)
	*p = 42
	fmt.Println("value at *p:", *p)
	zeroptr(p)
	fmt.Println("value at *p:", *p)
}
```

运行（指针地址每次不同）：

```bash
$ go run pointers.go
initial: 1
zeroval: 1
zeroptr: 0
pointer: 0x...
value at *p: 42
value at *p: 0
```

## `&` 取地址，`*` 解引用

```go
i := 1
p := &i      // p 是 *int，指向 i
fmt.Println(*p) // 1
*p = 2       // 通过指针修改 i
fmt.Println(i)  // 2
```

两个符号要分清：

- `&i`：取变量 `i` 的地址，得到 `*int`；
- `*p`：解引用指针，读取或写入它指向的值；
- `*int`：类型声明中的指针类型，表示“指向 int 的指针”。

`*` 在不同位置含义不同：类型里的 `*` 表示指针类型，表达式里的 `*` 表示解引用。

## 值传递与指针传递

Go 的参数永远是值传递。传入 `int` 时复制整数；传入 `*int` 时复制指针（地址），但指针指向的还是同一个变量。

```go
func zeroval(ival int) {
	ival = 0
}

func zeroptr(iptr *int) {
	*iptr = 0
}
```

调用：

```go
i := 1
zeroval(i)   // i 仍是 1
zeroptr(&i)  // i 变成 0
```

`zeroval` 修改的是副本；`zeroptr` 通过地址修改原变量。

需要注意：指针本身也是按值传递的。如果在函数内让参数指向另一个变量，调用方的指针不会改变：

```go
func reassign(p *int) {
	x := 100
	p = &x // 只修改了 p 这个副本
}
```

但通过 `*p` 修改指向的值，会影响调用方。

## nil 指针

指针的零值是 `nil`：

```go
var p *int
fmt.Println(p == nil) // true
```

解引用 nil 指针会 panic：

```go
fmt.Println(*p)
// panic: runtime error: invalid memory address or nil pointer dereference
```

使用指针前要判断：

```go
if p != nil {
	fmt.Println(*p)
}
```

nil 指针常用来表示“没有值”或“可选字段”，例如链表尾节点、树的孩子节点。

## 用 `new` 分配

`new(T)` 分配一个类型为 `T` 的零值变量，返回 `*T`：

```go
p := new(int)
fmt.Println(*p) // 0
*p = 42
fmt.Println(*p) // 42
```

它等价于：

```go
var x int
p := &x
```

对结构体也常用 `&T{}`：

```go
type Point struct{ X, Y int }
p := &Point{X: 1, Y: 2}
```

## 返回局部变量的指针是安全的

Go 是垃圾回收语言，返回局部变量的地址没有问题：

```go
func newInt() *int {
	x := 1
	return &x
}
```

编译器会做逃逸分析，把 `x` 分配到堆上；只要还有指针引用它，就不会被回收。这一点和 C/C++ 中返回局部变量地址是未定义行为完全不同。

## 结构体指针

访问结构体指针的字段时，Go 会自动解引用：

```go
type Person struct {
	Name string
	Age  int
}

p := &Person{Name: "Alice", Age: 30}
fmt.Println(p.Name) // 等价于 (*p).Name
p.Age = 31
```

方法调用也有类似的自动转换，后续方法一篇会展开。

## Go 指针的限制

- **没有指针运算**：不能写 `p++` 或 `p + 1`。
- **不能任意转换指针类型**：需要 `unsafe.Pointer` 才能做底层操作，普通代码不应使用。
- **不能取常量的地址**：

```go
const x = 1
p := &x
// invalid operation: cannot take address of x
```

- **不能取临时值的地址**（复合字面量除外）：

```go
p := &42
// invalid operation: cannot take address of 42
```

## 什么时候用指针

适合使用指针的场景：

- 函数需要修改调用方的变量；
- 结构体较大，复制成本高；
- 需要表示“可选”或“不存在”（nil）；
- 链表、树等需要互相引用的数据结构；
- 方法需要修改接收者（指针接收者）。

不需要指针的场景：

- 简单的小类型（int、bool、小结构体）；
- 只读访问，复制成本可以接受；
- 切片、map、channel 本身已经包含引用语义，通常不需要再取指针。

不要为了“看起来像 C”而到处使用指针。Go 的零值和值语义让很多代码不需要指针。

## 常见错误

### 1. 解引用 nil 指针

使用前判断 `p != nil`，或者确保指针一定被初始化。

### 2. 取常量或临时值的地址

`&1`、`&"x"` 都不合法。需要先赋给变量，或使用 `new`。

### 3. 以为指针参数一定能改变调用方指针

函数内的指针变量也是副本。`p = &x` 不会改变调用方的指针；`*p = x` 才会改变指向的值。

### 4. 在旧版本 Go 中保存循环变量地址

Go 1.21 及更早版本中，循环变量在迭代间复用，保存 `&i` 可能得到同一个地址。Go 1.22 起每次迭代都是新变量。

### 5. 对指针进行算术运算

Go 不支持 `p++`、`p + 1`。需要遍历数组时使用索引或切片。

### 6. 过度使用指针

小类型、只读参数通常直接传值更清晰。指针会增加 nil 检查和别名分析的成本。

## 和 Java 对照

| 概念 | Go | Java |
| --- | --- | --- |
| 显式指针 | `*T`、`&`、`*` | 没有 |
| 对象引用 | 结构体值或指针 | 对象引用 |
| 基本类型 | 值传递 | 值传递 |
| 修改调用方 | 传指针 | 通过对象引用修改字段 |
| 空值 | nil 指针 | null |
| 指针运算 | 不支持 | 不支持 |
| 内存管理 | GC | GC |

Java 的对象变量本质上也是“引用按值传递”，但没有显式的 `&` 和 `*`。Go 把值类型和指针类型区分得更清楚：`Person` 是值，`*Person` 是引用；Java 的 `Person` 变量通常已经是引用。

## 动手练习

1. 写 `zeroval` 和 `zeroptr`，验证值传递与指针传递的区别。
2. 用 `&` 取变量地址，通过 `*` 修改它。
3. 声明 nil 指针，判断后安全解引用。
4. 用 `new(int)` 分配指针，赋值并打印。
5. 写一个函数返回局部变量的指针，验证可以正常使用。
6. 用结构体指针修改字段，对比直接传结构体值的效果。

## 小结

- `&` 取地址，`*` 解引用，`*T` 是指针类型。
- Go 的参数按值传递；传指针才能修改调用方变量。
- 指针零值是 nil，解引用 nil 会 panic。
- `new(T)` 分配零值并返回指针；返回局部变量指针是安全的。
- 结构体指针访问字段会自动解引用。
- Go 没有指针运算，也不应随意使用 unsafe。

---

上一篇：[[Go/Advanced/Range Over Built-in Types|Go 进阶：range 遍历内置类型]]
下一篇：[[Go/Advanced/Strings and Runes|Go 进阶：字符串与 rune]]

返回 [[Go/Advanced/index|Go 进阶]]。
