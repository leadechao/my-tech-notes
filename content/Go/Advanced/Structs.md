---
title: "Go 进阶：结构体"
description: "从 Go by Example 的 Structs 示例出发，掌握结构体定义、字面量、零值、指针、字段访问、值语义、比较与匿名结构体。"
tags:
  - Go
  - 编程语言
  - 进阶
  - 结构体
created: 2026-10-29
modified: 2026-10-29
---

# Go 进阶：结构体

结构体把一组字段组合成一个记录。它是 Go 中构造复杂数据的主要方式，也是方法、接口、嵌入和泛型的基础。

和 Java 的类不同，Go 的结构体只负责数据布局；行为通过方法定义在类型上，组合通过嵌入实现，继承则不存在。

[Go by Example 的 Structs](https://gobyexample.com/structs) 覆盖了定义、字面量、指针和匿名结构体。本文补充值语义、比较、导出字段和常见错误。

## 完整示例

```go
package main

import "fmt"

type person struct {
	name string
	age  int
}

func newPerson(name string) *person {
	p := person{name: name}
	p.age = 42
	return &p
}

func main() {
	fmt.Println(person{"Bob", 20})
	fmt.Println(person{name: "Alice", age: 30})
	fmt.Println(person{name: "Fred"})
	fmt.Println(&person{name: "Ann", age: 40})
	fmt.Println(newPerson("Jon"))

	s := person{name: "Sean", age: 50}
	fmt.Println(s.name)

	s.age = 51
	fmt.Println(s)

	sp := &s
	sp.age = 52
	fmt.Println(sp.age)

	dog := struct {
		name   string
		isGood bool
	}{
		"Rex",
		true,
	}
	fmt.Println(dog)
}
```

运行：

```bash
$ go run structs.go
{Bob 20}
{Alice 30}
{Fred 0}
&{Ann 40}
&{Jon 42}
Sean
{Sean 51}
52
{Rex true}
```

## 定义结构体

```go
type person struct {
	name string
	age  int
}
```

`type` 定义一个新的命名类型，底层是结构体。字段名在前，类型在后；多个同类型字段可以合并：

```go
type point struct {
	x, y int
}
```

字段的零值遵循类型规则：`string` 是 `""`，`int` 是 `0`，指针是 `nil`。

## 结构体字面量

按字段顺序写值：

```go
p := person{"Bob", 20}
```

按字段名写值：

```go
p := person{name: "Alice", age: 30}
```

两种写法不能混用：

```go
p := person{"Bob", age: 20}
// mixture of field:value and value elements in struct literal
```

按位置初始化时必须提供所有字段；按字段名初始化可以省略字段，省略的字段取零值：

```go
p := person{name: "Fred"}
fmt.Println(p) // {Fred 0}
```

字段顺序发生变化时，按位置初始化的代码会悄悄错位，因此实际项目更推荐按字段名初始化。

## 指针与构造函数

`&person{...}` 返回结构体指针：

```go
p := &person{name: "Ann", age: 40}
fmt.Println(p) // &{Ann 40}
```

Go 没有构造函数语法，但习惯上会写一个 `newXxx` 函数：

```go
func newPerson(name string) *person {
	p := person{name: name}
	p.age = 42
	return &p
}
```

返回局部变量的指针是安全的，垃圾回收器会保证它存活到不再被引用。

## 字段访问与指针自动解引用

用点号访问字段：

```go
s := person{name: "Sean", age: 50}
fmt.Println(s.name) // Sean
s.age = 51
```

结构体指针也可以直接使用点号：

```go
sp := &s
sp.age = 52
fmt.Println(sp.age)
```

Go 会自动把 `sp.age` 解释为 `(*sp).age`，不需要手动解引用。

## 结构体是值类型

结构体赋值会复制所有字段：

```go
a := person{name: "Alice", age: 30}
b := a
b.age = 31

fmt.Println(a.age) // 30
fmt.Println(b.age) // 31
```

把结构体传给函数同样会复制：

```go
func birthday(p person) {
	p.age++
}

birthday(a)
fmt.Println(a.age) // 不变
```

需要修改原结构体或避免复制大结构体时，使用指针：

```go
func birthdayPtr(p *person) {
	p.age++
}

birthdayPtr(&a)
```

## 结构体比较

如果所有字段都可比较，结构体可以用 `==` 和 `!=` 比较：

```go
p1 := person{name: "Alice", age: 30}
p2 := person{name: "Alice", age: 30}
fmt.Println(p1 == p2) // true
```

包含切片、map、函数等不可比较字段的结构体不能比较：

```go
type bad struct {
	items []int
}
// bad{} == bad{} // invalid operation
```

## 匿名结构体

如果结构体只使用一次，可以不给类型命名：

```go
dog := struct {
	name   string
	isGood bool
}{
	"Rex",
	true,
}
fmt.Println(dog) // {Rex true}
```

匿名结构体常用于表驱动测试、临时 JSON 数据和一次性配置。字段名和类型完全相同（包括标签）的匿名结构体类型是相同的，但不要依赖这个特性做复杂设计。

## 导出字段与标签

字段名首字母大写时，其他包可以访问；小写字段只在当前包可见：

```go
type User struct {
	Name string // 导出
	age  int    // 未导出
}
```

结构体标签常用于 JSON、数据库映射等：

```go
type User struct {
	Name string `json:"name"`
	Age  int    `json:"age,omitempty"`
}
```

标签本身是字符串，具体含义由使用它的库解释。Go 编译器不会校验标签内容。

## 常见错误

### 1. 混用按位置和按字段名初始化

```go
p := person{"Bob", age: 20}
// mixture of field:value and value elements
```

要么全部按位置，要么全部按字段名。

### 2. 按位置初始化时字段顺序变化

字段顺序改变后，旧代码可能把值赋给错误的字段。优先使用字段名。

### 3. 以为结构体赋值共享数据

结构体是值类型，赋值会复制。需要共享时使用指针。

### 4. 比较包含不可比较字段的结构体

切片、map、函数不能比较，包含它们的结构体也不能用 `==`。

### 5. 忘记导出字段

其他包需要访问的字段必须首字母大写。JSON 等库通常也只能看到导出字段。

### 6. 复制包含锁的结构体

`sync.Mutex` 等类型不应复制。包含锁的结构体应通过指针传递，避免复制后锁状态不一致。

## 和 Java 对照

| 概念 | Go | Java |
| --- | --- | --- |
| 数据定义 | `struct` | `class` |
| 方法 | 定义在类型外 | 定义在类内 |
| 继承 | 不支持，用组合/嵌入 | 支持 `extends` |
| 值语义 | 结构体赋值复制 | 对象赋值复制引用 |
| 零值 | 字段自动取零值 | 字段有默认值，局部变量需初始化 |
| 访问控制 | 首字母大小写 | `public` / `private` |
| 标签 | struct tag | 注解（Annotation） |

Java 的类同时包含字段和方法，Go 把数据和行为分开：结构体定义数据，方法定义行为。Java 对象通常是引用，Go 结构体默认是值，这个差异会直接影响函数参数和赋值行为。

## 动手练习

1. 定义一个 `Book` 结构体，包含标题、作者和价格字段。
2. 分别用位置和字段名两种方式初始化。
3. 写一个构造函数，返回 `*Book` 并设置默认值。
4. 修改结构体副本的字段，验证原结构体不变。
5. 比较两个字段完全相同的结构体。
6. 定义匿名结构体保存一组键值配置。
7. 给结构体添加 JSON 标签，用 `encoding/json` 序列化。

## 小结

- 结构体把多个字段组合成一个记录。
- 字面量支持按位置或按字段名初始化，二者不能混用。
- 省略字段取零值；指针用 `&T{}` 或构造函数创建。
- 结构体是值类型，赋值和传参都会复制。
- 所有字段可比较时，结构体可以用 `==` 比较。
- 导出字段首字母大写；标签由使用它的库解释。

---

上一篇：[[Go/Advanced/Strings and Runes|Go 进阶：字符串与 rune]]
下一篇：[[Go/Advanced/Methods|Go 进阶：方法]]

返回 [[Go/Advanced/index|Go 进阶]]。
