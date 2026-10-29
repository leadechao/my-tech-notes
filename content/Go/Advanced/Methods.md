---
title: "Go 进阶：方法"
description: "从 Go by Example 的 Methods 示例出发，理解接收者、值接收者与指针接收者、方法集、自动解引用，以及方法对接口实现的影响。"
tags:
  - Go
  - 编程语言
  - 进阶
  - 方法
created: 2026-10-29
modified: 2026-10-29
---

# Go 进阶：方法

方法是带接收者的函数。它把行为绑定到某个类型上，让结构体拥有类似对象的方法调用语法。

Go 没有类，也没有继承。方法和结构体是分开定义的，但通过接收者建立关联。理解值接收者和指针接收者的区别，是理解接口实现的关键。

[Go by Example 的 Methods](https://gobyexample.com/methods) 用矩形面积和周长演示了两种接收者。本文补充方法集、可变性、命名类型和常见陷阱。

## 完整示例

```go
package main

import "fmt"

type rect struct {
	width, height int
}

func (r *rect) area() int {
	return r.width * r.height
}

func (r rect) perim() int {
	return 2*r.width + 2*r.height
}

func main() {
	r := rect{width: 10, height: 5}

	fmt.Println("area: ", r.area())
	fmt.Println("perim:", r.perim())

	rp := &r
	fmt.Println("area: ", rp.area())
	fmt.Println("perim:", rp.perim())
}
```

运行：

```bash
$ go run methods.go
area:  50
perim: 30
area:  50
perim: 30
```

## 接收者

方法声明在 `func` 和函数名之间多了一个接收者：

```go
func (r *rect) area() int {
	return r.width * r.height
}
```

`(r *rect)` 表示接收者 `r` 的类型是 `*rect`。调用时：

```go
r.area()
```

接收者就像是其他语言里的 `this`，但它是显式写出来的，名字可以自选。惯例是使用类型名首字母的小写，例如 `r`、`c`。

## 值接收者与指针接收者

```go
func (r *rect) area() int {
	return r.width * r.height
}

func (r rect) perim() int {
	return 2*r.width + 2*r.height
}
```

区别：

- **值接收者** `r rect`：方法拿到结构体副本，修改 `r` 不会影响原值；
- **指针接收者** `r *rect`：方法拿到指针，可以通过 `r` 修改原值。

```go
func (r rect) setWidth(w int) {
	r.width = w // 只修改副本
}

func (r *rect) setWidthPtr(w int) {
	r.width = w // 修改原值
}
```

需要修改接收者、或者结构体较大需要避免复制时，使用指针接收者。只读的小结构体可以用值接收者。

## 自动取地址和解引用

Go 会在方法调用时自动处理值和指针：

```go
r := rect{width: 10, height: 5}
r.area()   // 等价于 (&r).area()
```

即使 `area` 是指针接收者，只要 `r` 是可寻址的变量，就可以直接调用。

反过来，指针也可以调用值接收者方法：

```go
rp := &r
rp.perim() // 等价于 (*rp).perim()
```

但自动取地址要求值是可寻址的。map 元素、函数返回值等不可寻址的值，不能直接调用指针接收者方法。

## 方法集与接口

方法集决定了一个类型能实现哪些接口：

- 类型 `T` 的方法集包含所有**值接收者**方法；
- 类型 `*T` 的方法集包含所有**值接收者和指针接收者**方法。

因此，如果接口方法由指针接收者实现，只有 `*T` 满足接口，`T` 不满足：

```go
type Counter struct{ n int }

func (c *Counter) Inc() {
	c.n++
}

type Incer interface {
	Inc()
}

var a Incer = &Counter{} // 正确
// var b Incer = Counter{} // 错误：Counter 没有实现 Inc
```

报错：

```text
cannot use Counter{} ... as Incer value: Counter does not implement Incer (method Inc has pointer receiver)
```

这是 Go 初学者最常见的接口实现问题之一。遇到“方法明明写了却说不满足接口”，先检查接收者是值还是指针。

## 方法可以定义在任意命名类型上

方法不限于结构体，可以定义在任何当前包内定义的命名类型上：

```go
type MyInt int

func (m MyInt) Double() MyInt {
	return m * 2
}

fmt.Println(MyInt(3).Double()) // 6
```

但不能直接给 `int`、`[]int`、`map[string]int` 这类非命名类型定义方法，也不能给其他包的类型定义方法：

```go
// func (i int) Double() int // 错误
```

需要时先定义自己的命名类型。

## 方法与函数

方法本质上就是带接收者的函数。下面两种写法等价：

```go
r.area()
(*rect).area(&r) // 方法表达式
```

方法值可以赋值给变量：

```go
f := r.area
fmt.Println(f())
```

如果方法是指针接收者，方法值会捕获接收者的指针，之后调用仍然操作同一个对象。

## nil 接收者

指针接收者方法可以处理 nil 接收者：

```go
func (c *Counter) Inc() {
	if c == nil {
		return
	}
	c.n++
}
```

调用 `var c *Counter; c.Inc()` 不会 panic，因为方法内部先判断了 nil。但如果方法直接访问 `c.n`，就会 panic。

值接收者方法不能处理 nil，因为调用前需要解引用指针，nil 会直接 panic。

## 常见错误

### 1. 值接收者修改字段无效

```go
func (r rect) setWidth(w int) {
	r.width = w
}
```

修改的是副本。需要指针接收者。

### 2. 指针接收者导致接口不满足

`T` 的方法集不包含指针接收者方法。接口变量需要赋 `&T{}`，而不是 `T{}`。

### 3. 对不可寻址的值调用指针方法

map 元素、函数返回值等不可寻址，不能自动取地址。先把值取到变量里，再调用。

### 4. 给非命名类型定义方法

`int`、`[]int` 等不是当前包定义的命名类型，不能直接加方法。先 `type MyInt int`。

### 5. 方法与字段重名

字段和方法不能同名。编译器会报 `field and method with the same name`。

### 6. 在值接收者中复制大结构体

值接收者每次调用都会复制结构体。大对象应使用指针接收者。

## 和 Java 对照

| 概念 | Go | Java |
| --- | --- | --- |
| 方法定义 | 类型外，带接收者 | 类内部 |
| `this` | 显式接收者 | 隐式 `this` |
| 值/引用接收者 | 值接收者或指针接收者 | 对象引用 |
| 继承 | 不支持 | 支持 |
| 重写 | 通过接口/嵌入，不是继承重写 | `@Override` |
| 方法重载 | 不支持 | 支持 |
| 接口实现 | 隐式，满足方法集即可 | 显式 `implements` |

Java 的方法天然操作对象引用，`this` 指向当前对象；Go 需要显式选择值接收者还是指针接收者。这个选择会直接影响能否修改原值以及是否满足接口。

## 动手练习

1. 定义 `rect`，分别用值接收者和指针接收者实现面积方法。
2. 写一个修改字段的方法，先用值接收者观察失败，再改成指针接收者。
3. 用 `&r` 和 `r` 分别调用指针/值接收者方法，验证自动转换。
4. 定义接口 `Incer`，让 `Counter` 通过指针接收者实现它，观察 `Counter{}` 和 `&Counter{}` 的差异。
5. 给自定义的 `MyInt` 类型添加 `Double` 方法。
6. 写一个处理 nil 接收者的方法，安全地返回默认值。

## 小结

- 方法是带接收者的函数，接收者写在 `func` 和函数名之间。
- 值接收者拿到副本，指针接收者可以修改原值。
- Go 在方法调用时自动取地址/解引用，但要求值可寻址。
- `T` 的方法集只包含值接收者方法；`*T` 包含两者。
- 接口实现取决于方法集，指针接收者常导致 `T` 不满足接口。
- 方法可以定义在任何当前包的命名类型上。

---

上一篇：[[Go/Advanced/Structs|Go 进阶：结构体]]
下一篇：[[Go/Advanced/Interfaces|Go 进阶：接口]]

返回 [[Go/Advanced/index|Go 进阶]]。
