---
title: "Go 进阶：多返回值"
description: "从 Go by Example 的 Multiple Return Values 示例出发，掌握多返回值、空白标识符、命名返回值，以及 result/error 的惯用写法。"
tags:
  - Go
  - 编程语言
  - 进阶
  - 函数
created: 2026-10-29
modified: 2026-10-29
---

# Go 进阶：多返回值

Go 原生支持函数返回多个值。这不是语法糖，而是语言层面的特性，最典型的用途是一次返回“结果 + 错误”。

[Go by Example 的 Multiple Return Values](https://gobyexample.com/multiple-return-values) 演示了两个返回值的函数。本文补充命名返回值、错误处理惯例和多返回值的使用限制。

## 完整示例

```go
package main

import "fmt"

func vals() (int, int) {
	return 3, 7
}

func main() {
	a, b := vals()
	fmt.Println(a)
	fmt.Println(b)

	_, c := vals()
	fmt.Println(c)
}
```

运行：

```bash
$ go run multiple-return-values.go
3
7
7
```

## 返回多个值

返回值列表写在参数列表之后：

```go
func vals() (int, int) {
	return 3, 7
}
```

`(int, int)` 表示返回两个 `int`。调用时用多重赋值接收：

```go
a, b := vals()
```

返回值按位置对应，第一个返回值赋给 `a`，第二个赋给 `b`。

Go 没有“元组”类型。多返回值是一个特殊的多值表达式，不能直接存进一个变量：

```go
a := vals()
// assignment mismatch: 1 variable but vals returns 2 values
```

## 用 `_` 忽略不需要的返回值

如果只关心其中一个值，可以用空白标识符 `_` 丢弃其他值：

```go
_, c := vals()
fmt.Println(c) // 7
```

`_` 不能被读取，也不能参与运算。它的作用是明确表示“这个值我不需要”。

## 命名返回值

返回值可以在签名中命名：

```go
func divmod(a, b int) (q, r int) {
	q = a / b
	r = a % b
	return
}
```

`q` 和 `r` 在函数体内就像已经声明的变量。最后的 `return` 不带参数，称为“裸返回”（naked return），会返回当前 `q` 和 `r` 的值。

命名返回值适合短函数，能让签名更清晰；但在长函数中，裸返回会让读者难以一眼看出返回了什么。多数情况下，显式写 `return q, r` 更清楚。

命名返回值还可以在 `defer` 中修改，这是某些错误处理技巧的基础，但初学阶段不建议滥用。

## 惯用写法：结果 + 错误

Go 没有异常，错误通常作为最后一个返回值：

```go
func div(a, b int) (int, error) {
	if b == 0 {
		return 0, errors.New("division by zero")
	}
	return a / b, nil
}
```

调用方必须显式检查：

```go
q, err := div(10, 2)
if err != nil {
	fmt.Println("failed:", err)
	return
}
fmt.Println(q)
```

约定俗成的规则：

- 错误放在最后一个返回值；
- 成功时错误为 `nil`；
- 失败时结果返回零值；
- 不要用 `_` 忽略错误，除非你非常确定它不可能发生或无关紧要。

这个模式在标准库中随处可见，例如 `os.Open`、`strconv.Atoi`、`json.Marshal`。

## 多返回值直接传给函数

如果多返回值的数量和类型正好匹配另一个函数的参数，可以直接传递：

```go
fmt.Println(vals()) // 3 7
```

这是 Go 的特例，等价于 `fmt.Println(3, 7)`。但不能和其他参数混用：

```go
fmt.Println(vals(), 1)
// multiple-value vals() (value of type (int, int)) in single-value context
```

多返回值也不能直接参与算术、比较或赋值给单个变量。

## 常见错误

### 1. 接收变量数量不匹配

```go
a := vals()
// assignment mismatch: 1 variable but vals returns 2 values
```

必须用两个变量接收，或用 `_` 丢弃其中一个。

### 2. 忽略错误

```go
q, _ := div(10, 0)
fmt.Println(q) // 0，错误被吞掉了
```

忽略错误会让调用方无法知道失败原因。除非有明确理由，否则应检查 `err`。

### 3. 在长函数中滥用裸返回

命名返回值加裸返回在短函数中很简洁，但函数变长后可读性会下降。显式 `return` 通常更安全。

### 4. 以为多返回值是元组

Go 没有元组类型，不能把 `vals()` 存成一个值，也不能把多返回值放进切片或 map。

### 5. 错误没有放在最后

虽然语言不强制，但把错误放在最后是 Go 的惯例。调用方和工具都依赖这个约定。

## 和 Java 对照

| 概念 | Go | Java |
| --- | --- | --- |
| 多返回值 | 原生支持 | 不支持，需要封装对象/数组 |
| 错误处理 | 返回 `error` | 异常 `throw/catch` |
| 忽略返回值 | `_` | 直接不接收（单返回值） |
| 命名返回值 | 支持 | 不支持 |
| 元组 | 没有 | 没有内置元组 |

Java 方法只能返回一个值，要返回多个结果通常需要定义 record/class，或使用 `Map.Entry`、数组等容器。Go 的多返回值让“结果 + 错误”写起来非常自然。

## 动手练习

1. 写一个返回两个 `int` 的函数，分别用两个变量和 `_` 接收。
2. 写一个 `divmod` 函数，返回商和余数。
3. 用命名返回值改写 `divmod`，再改成显式 `return`，比较可读性。
4. 写一个返回 `(int, error)` 的除法函数，处理除零错误。
5. 尝试用单个变量接收两个返回值，观察编译错误。
6. 尝试把多返回值和其他参数一起传给 `fmt.Println`，观察错误。

## 小结

- Go 函数可以返回多个值，调用时用多重赋值接收。
- `_` 用于丢弃不需要的返回值。
- 命名返回值可以使用裸返回，但长函数中应谨慎。
- “结果 + error”是 Go 最重要的惯用模式，错误放在最后。
- 多返回值不是元组，不能当作单个值使用。

---

上一篇：[[Go/Advanced/Functions|Go 进阶：函数]]
下一篇：[[Go/Advanced/Variadic Functions|Go 进阶：变参函数]]

返回 [[Go/Advanced/index|Go 进阶]]。
