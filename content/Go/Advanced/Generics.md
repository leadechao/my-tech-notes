---
title: "Go 进阶：泛型"
description: "从 Go by Example 的 Generics 示例出发，理解类型参数、约束、comparable、~ 底层类型、泛型类型与类型推断。"
tags:
  - Go
  - 编程语言
  - 进阶
  - 泛型
created: 2026-10-29
modified: 2026-10-29
---

# Go 进阶：泛型

Go 1.18 引入了泛型（类型参数）。它让我们可以写出对多种类型都适用、同时保留编译期类型检查的代码，而不必依赖 `any` 和类型断言。

泛型的核心是**类型参数**和**约束**：类型参数代表未知类型，约束规定这个类型必须支持哪些操作。

[Go by Example 的 Generics](https://gobyexample.com/generics) 给出了泛型函数和泛型类型的例子。本文补充约束、`~`、类型推断、零值和常见错误。

## 完整示例

```go
package main

import "fmt"

func SlicesIndex[S ~[]E, E comparable](s S, v E) int {
	for i := range s {
		if v == s[i] {
			return i
		}
	}
	return -1
}

type List[T any] struct {
	head, tail *element[T]
}

type element[T any] struct {
	next *element[T]
	val  T
}

func (lst *List[T]) Push(v T) {
	if lst.tail == nil {
		lst.head = &element[T]{val: v}
		lst.tail = lst.head
	} else {
		lst.tail.next = &element[T]{val: v}
		lst.tail = lst.tail.next
	}
}

func (lst *List[T]) AllElements() []T {
	var elems []T
	for e := lst.head; e != nil; e = e.next {
		elems = append(elems, e.val)
	}
	return elems
}

func main() {
	var s = []string{"foo", "bar", "zoo"}
	fmt.Println("index of zoo:", SlicesIndex(s, "zoo"))
	_ = SlicesIndex[[]string, string](s, "zoo")

	lst := List[int]{}
	lst.Push(10)
	lst.Push(13)
	lst.Push(23)
	fmt.Println("list:", lst.AllElements())
}
```

运行：

```bash
$ go run generics.go
index of zoo: 2
list: [10 13 23]
```

## 泛型函数

```go
func SlicesIndex[S ~[]E, E comparable](s S, v E) int {
	for i := range s {
		if v == s[i] {
			return i
		}
	}
	return -1
}
```

`[S ~[]E, E comparable]` 是类型参数列表：

- `S` 是切片类型；
- `E` 是元素类型；
- `~[]E` 表示“底层类型为 `[]E` 的类型”；
- `comparable` 表示 `E` 可以用 `==` 和 `!=` 比较。

函数体里，`s` 的类型是 `S`，`v` 的类型是 `E`，编译器知道 `S` 是元素为 `E` 的切片，因此可以 `range` 和索引。

## 约束

约束是类型参数的“类型范围”。常见约束：

### `any`

```go
func Print[T any](v T) {
	fmt.Println(v)
}
```

`any` 表示任意类型，等价于 `interface{}`。但 `T` 仍然是静态类型，不能在函数体里对它做任意操作。

### `comparable`

```go
func Contains[T comparable](xs []T, target T) bool {
	for _, x := range xs {
		if x == target {
			return true
		}
	}
	return false
}
```

只有 `comparable` 约束的类型才能使用 `==`、`!=`，也才能作为 map 的键。

### 联合约束

```go
type Number interface {
	~int | ~int64 | ~float64
}

func Sum[T Number](xs []T) T {
	var total T
	for _, x := range xs {
		total += x
	}
	return total
}
```

`|` 表示“或”，`~` 表示底层类型。`Number` 约束允许所有底层类型为 `int`、`int64` 或 `float64` 的类型，包括 `type MyInt int` 这样的命名类型。

标准库中也有类似约束，例如 `cmp.Ordered`（Go 1.21+）覆盖所有可排序的基本类型。

### 带方法的约束

约束本身是接口，可以包含方法：

```go
type Stringer interface {
	String() string
}

func PrintAll[T Stringer](xs []T) {
	for _, x := range xs {
		fmt.Println(x.String())
	}
}
```

这要求 `T` 实现 `String()` 方法。

## `~`：底层类型

`~T` 表示“底层类型为 `T` 的所有类型”。

```go
type MyInt int

type IntLike interface {
	~int
}
```

`int` 和 `MyInt` 都满足 `IntLike`；如果约束写成 `int`，只有 `int` 本身满足，`MyInt` 不满足。

切片的例子：

```go
type MySlice []int

func Index[S ~[]E, E comparable](s S, v E) int
```

`~[]E` 让 `MySlice` 也能传入。如果写成 `S []E`，`MySlice` 不满足约束：

```text
MySlice does not satisfy []int (possibly missing ~ for []int)
```

这是泛型约束里最常见的错误之一。

## 泛型类型

```go
type List[T any] struct {
	head, tail *element[T]
}

type element[T any] struct {
	next *element[T]
	val  T
}
```

`List[T]` 是一个泛型类型，使用时必须实例化：

```go
lst := List[int]{}
lst.Push(10)
lst.Push(13)
```

方法接收者必须带上类型参数：

```go
func (lst *List[T]) Push(v T) {
	// ...
}
```

方法体里的 `T` 与外层 `List[T]` 的类型参数一致。不同类型实例化会得到不同的具体类型，例如 `List[int]` 和 `List[string]`。

## 类型推断

调用泛型函数时，编译器通常能从参数推断类型：

```go
SlicesIndex(s, "zoo")
```

这里 `s` 是 `[]string`，`"zoo"` 是 `string`，编译器推断 `S = []string`、`E = string`。

也可以显式指定：

```go
SlicesIndex[[]string, string](s, "zoo")
```

显式实例化在类型无法推断、或需要明确指定类型时有用。日常调用应优先依赖推断，代码更简洁。

## 零值与操作限制

泛型函数里可以声明 `T` 的零值：

```go
func Zero[T any]() T {
	var zero T
	return zero
}
```

但不能对 `T` 做约束之外的操作：

```go
func Bad[T any](a, b T) bool {
	return a == b
}
// invalid operation: a == b (incomparable types in type set)
```

因为 `any` 不保证可比较。需要比较时加 `comparable` 约束；需要算术时加数值约束。

## 泛型与接口如何选择

- 需要**运行时多态**、一组类型共享行为：用接口。
- 需要**编译期类型安全**、对多种类型复用同一算法：用泛型。
- 只是存储任意类型、后续还要断言：考虑 `any`，但通常不是最佳设计。
- 类型集合固定且需要运算符：用泛型约束，避免 `any` + 断言。

泛型不是接口的替代品。接口表达“能做什么”，泛型表达“对任意满足约束的类型，算法都一样”。

## 常见错误

### 1. 约束太宽，操作不合法

`T any` 不能比较、不能做算术。根据需要的操作选择 `comparable` 或自定义约束。

### 2. 忘记 `~`

`S []E` 不接受命名切片类型。需要兼容命名类型时写 `S ~[]E`。

### 3. 把 `any` 当成动态类型

`T any` 仍然是静态类型，不能在函数体里随便调用方法或做类型断言。需要断言时先转成 `any` 再处理。

### 4. 类型推断失败

有时编译器无法从参数推断所有类型参数，需要显式实例化，或调整函数签名。

### 5. 过度泛化

不是所有函数都需要泛型。只为一种类型服务的代码，用具体类型更清晰。泛型会增加签名复杂度和编译负担。

### 6. 忽略约束的方法集

约束接口包含的方法必须全部实现。指针接收者方法会影响类型是否满足约束，和方法集的规则一致。

## 和 Java 对照

| 概念 | Go | Java |
| --- | --- | --- |
| 类型参数 | `func F[T any]` | `<T> T f()` |
| 约束 | 接口 + 联合 + `~` | `extends` / `super` 边界 |
| 基本类型 | 可以直接作为类型参数 | 需要包装类型（泛型不支持基本类型） |
| 类型擦除 | 不做传统类型擦除 | 运行时擦除 |
| 运算符约束 | 通过类型集约束 | 不能约束运算符 |
| 泛型方法 | 支持（版本相关） | 支持 |
| 通配符 | 无直接等价 | `? extends T` / `? super T` |

Java 泛型在运行时会被擦除，不能直接用 `int` 作为类型参数；Go 的泛型保留类型信息，可以直接实例化 `List[int]`。Java 用边界表达约束，Go 用接口类型集表达约束，能力更强但也更复杂。

## 动手练习

1. 写一个泛型 `Contains[T comparable](xs []T, target T) bool`。
2. 用 `Number` 约束实现 `Sum`，分别对 `[]int` 和 `[]float64` 求和。
3. 定义 `type MyInt int`，验证 `~int` 约束能接受它。
4. 用 `List[T]` 存储 `string`，实现 `Push` 和 `AllElements`。
5. 尝试在 `T any` 上使用 `==`，观察编译错误，再加 `comparable` 修复。
6. 用 `slices.Index` 替代手写的 `SlicesIndex`，比较两者签名。

## 小结

- 泛型用类型参数和约束实现类型安全的复用。
- `any` 表示任意类型，`comparable` 允许比较，联合约束和 `~` 可精确描述类型集合。
- `~T` 匹配底层类型为 `T` 的命名类型。
- 泛型类型使用时需要实例化，方法接收者要带类型参数。
- 类型推断通常能自动完成，必要时可显式实例化。
- 泛型适合算法和容器，接口适合运行时多态，二者互补。

---

上一篇：[[Go/Advanced/Struct Embedding|Go 进阶：结构体嵌入]]
下一篇：[Go by Example: Range over Iterators](https://gobyexample.com/range-over-iterators)

返回 [[Go/Advanced/index|Go 进阶]]。
