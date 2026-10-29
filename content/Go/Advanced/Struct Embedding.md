---
title: "Go 进阶：结构体嵌入"
description: "从 Go by Example 的 Struct Embedding 示例出发，理解匿名字段、字段与方法提升、接口实现、方法遮蔽与组合优于继承。"
tags:
  - Go
  - 编程语言
  - 进阶
  - 嵌入
created: 2026-10-29
modified: 2026-10-29
---

# Go 进阶：结构体嵌入

Go 没有继承，但提供了结构体嵌入（embedding）。把一个类型匿名地放进另一个结构体，它的字段和方法会被“提升”到外层，调用时可以像访问自己的成员一样访问。

嵌入常被用来实现组合和接口复用，但它不是继承：没有虚函数，没有子类替换，也没有方法重写。理解这一点，才能避免把 Java 的继承思维直接搬过来。

[Go by Example 的 Struct Embedding](https://gobyexample.com/struct-embedding) 用 `base` 和 `container` 演示了字段与方法提升。本文补充歧义、遮蔽、指针嵌入和设计建议。

## 完整示例

```go
package main

import "fmt"

type base struct {
	num int
}

func (b base) describe() string {
	return fmt.Sprintf("base with num=%v", b.num)
}

type container struct {
	base
	str string
}

func main() {
	co := container{
		base: base{
			num: 1,
		},
		str: "some name",
	}

	fmt.Printf("co={num: %v, str: %v}\n", co.num, co.str)
	fmt.Println("also num:", co.base.num)
	fmt.Println("describe:", co.describe())

	type describer interface {
		describe() string
	}

	var d describer = co
	fmt.Println("describer:", d.describe())
}
```

运行：

```bash
$ go run struct-embedding.go
co={num: 1, str: some name}
also num: 1
describe: base with num=1
describer: base with num=1
```

## 嵌入的写法

```go
type container struct {
	base
	str string
}
```

`base` 是一个**匿名字段**：没有字段名，只有类型名。它的字段名默认就是 `base`，所以初始化时可以写：

```go
co := container{
	base: base{num: 1},
	str:  "some name",
}
```

嵌入不是“继承一个父类”，而是“把 `base` 作为 `container` 的一个字段，并允许直接访问它的导出成员”。

## 字段提升

`base` 的字段 `num` 被提升到 `container`：

```go
co.num        // 等价于 co.base.num
co.base.num   // 显式路径
```

字段提升只是语法便利，底层仍然是嵌套结构。`container` 的内存布局包含一个完整的 `base`。

## 方法提升

`base` 的方法也会被提升：

```go
fmt.Println(co.describe()) // 等价于 co.base.describe()
```

如果 `container` 没有自己的 `describe` 方法，`co.describe()` 会调用嵌入字段的方法。

方法提升让嵌入类型可以“免费”获得一组行为。标准库中常见这种用法，例如在自定义类型中嵌入 `sync.Mutex` 或 `bytes.Buffer`。

## 通过嵌入实现接口

```go
type describer interface {
	describe() string
}

var d describer = co
```

`container` 没有显式定义 `describe`，但因为嵌入了 `base`，方法被提升，所以 `container` 满足 `describer` 接口。

这是嵌入最有价值的用途之一：把一组方法复用给多个类型，而不需要写重复的转发代码。

## 遮蔽与“重写”

如果外层类型定义了自己的同名方法，它会遮蔽嵌入类型的方法：

```go
type Base struct{}

func (Base) Name() string {
	return "base"
}

type Derived struct {
	Base
}

func (Derived) Name() string {
	return "derived"
}
```

调用：

```go
d := Derived{}
fmt.Println(d.Name())      // derived
fmt.Println(d.Base.Name()) // base
```

这不是面向对象意义上的“重写”。`Derived.Name` 只是外层的方法，`Base.Name` 仍然存在。把 `Derived` 赋给一个要求 `Name()` 的接口时，调用的是 `Derived.Name`；但通过 `d.Base` 调用仍然是 `Base.Name`。

Go 没有虚方法表，也没有“父类引用指向子类对象”的动态派发。需要多态时，应使用接口。

## 字段或方法歧义

如果两个嵌入类型有同名字段或方法，直接访问会产生歧义：

```go
type A struct{ X int }
type B struct{ X int }

type C struct {
	A
	B
}

var c C
fmt.Println(c.X)
// ambiguous selector c.X
```

必须显式指定路径：

```go
fmt.Println(c.A.X)
fmt.Println(c.B.X)
```

方法歧义同理。嵌入层次越深，歧义越难发现，所以应保持嵌入结构简单。

## 嵌入指针

可以嵌入指针类型：

```go
type container struct {
	*base
	str string
}
```

初始化时需要提供指针：

```go
co := container{
	base: &base{num: 1},
	str:  "name",
}
```

字段和方法仍然可以提升。但要小心 nil：如果嵌入的指针是 nil，访问提升的字段或调用方法可能 panic。

```go
var co container
fmt.Println(co.num) // panic: nil pointer dereference
```

嵌入值类型（`base`）不会有这个问题，零值字段是可用的。

## 嵌入接口

结构体也可以嵌入接口：

```go
type ReadWriter struct {
	Reader
	Writer
}
```

这样 `ReadWriter` 就拥有了 `Reader` 和 `Writer` 的方法集。它适合组合接口能力，但调用前必须确保嵌入的接口字段已经赋了具体实现，否则会 nil panic。

接口本身也可以嵌入接口：

```go
type ReadWriter interface {
	Reader
	Writer
}
```

这是接口组合，不是结构体嵌入。

## 嵌入与 JSON

匿名嵌入字段在 JSON 编码时通常会被“扁平化”：

```go
type Base struct {
	ID int `json:"id"`
}

type User struct {
	Base
	Name string `json:"name"`
}
```

`User` 编码后通常得到 `{"id":1,"name":"..."}`，而不是嵌套的 `base` 对象。具体行为取决于字段是否导出、标签和编码库版本，实际项目中应写测试确认。

## 常见错误

### 1. 把嵌入当成继承

Go 没有继承，也没有虚方法。嵌入只是字段和方法的提升，不能替代接口多态。

### 2. 以为外层方法会“覆盖”内层

外层同名方法只是遮蔽，内层方法仍然可以通过显式路径调用。不要把二者当成重写关系。

### 3. 两个嵌入类型成员重名

直接访问会报 `ambiguous selector`。需要显式指定 `c.A.X` 或 `c.B.X`。

### 4. 嵌入 nil 指针

嵌入 `*T` 且没有初始化时，访问提升字段会 panic。需要确保指针非 nil，或嵌入值类型。

### 5. 嵌入包含锁的结构体并复制

`sync.Mutex` 等类型不应复制。嵌入锁的结构体应通过指针使用，避免复制导致锁状态分裂。

### 6. 误以为嵌入接口字段已经初始化

嵌入接口只是声明字段，零值是 nil。调用前必须赋具体实现。

## 和 Java 对照

| 概念 | Go | Java |
| --- | --- | --- |
| 代码复用 | 嵌入/组合 | 继承/组合 |
| 方法提升 | 支持 | 继承方法 |
| 方法重写 | 无虚方法，只有遮蔽 | `@Override` 动态派发 |
| 多态 | 接口 | 继承/接口 |
| 字段访问 | 提升 + 显式路径 | 继承字段 |
| 构造 | 字面量/构造函数 | 构造器 |
| 类型关系 | 没有子类型关系 | 子类是父类的子类型 |

Java 的继承会建立“is-a”关系和虚方法派发；Go 的嵌入只是“has-a + 方法提升”。需要“is-a”语义时，用接口表达。

## 动手练习

1. 定义 `base` 和 `container`，嵌入并访问提升字段。
2. 给 `base` 添加方法，通过 `container` 调用。
3. 定义接口 `describer`，验证 `container` 通过嵌入满足接口。
4. 定义两个都带 `X` 字段的结构体，嵌入到第三个结构体，观察歧义错误并显式访问。
5. 给外层类型定义同名方法，验证遮蔽和显式调用内层方法。
6. 嵌入 `*base`，分别用 nil 和非 nil 指针访问字段，观察 panic。
7. 用嵌入组合两个接口，赋一个具体实现并调用。

## 小结

- 嵌入是匿名字段，字段名默认是类型名。
- 嵌入类型的字段和方法会被提升到外层。
- 嵌入可以让外层类型自动满足接口。
- 外层同名方法只是遮蔽，不是继承重写。
- 多个嵌入类型成员重名时会产生歧义，需要显式路径。
- 嵌入指针要防止 nil；嵌入接口要确保已初始化。
- 嵌入是组合工具，不建立子类型关系。

---

上一篇：[[Go/Advanced/Enums|Go 进阶：枚举]]
下一篇：[[Go/Advanced/Generics|Go 进阶：泛型]]

返回 [[Go/Advanced/index|Go 进阶]]。
