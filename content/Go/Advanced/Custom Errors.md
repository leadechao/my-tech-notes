---
title: "Go 进阶：自定义错误"
description: "从 Go by Example 的 Custom Errors 示例出发，定义带字段的错误类型，用 errors.As/AsType 提取错误，并理解指针接收者与 typed nil 陷阱。"
tags:
  - Go
  - 编程语言
  - 进阶
  - 错误处理
created: 2026-10-29
modified: 2026-10-29
---

# Go 进阶：自定义错误

`errors.New` 只能携带一段文本。如果调用方需要知道“哪个参数错了”“错误码是多少”“哪个字段校验失败”，就需要自定义错误类型。

在 Go 中，自定义错误就是实现了 `Error() string` 的类型。它可以带字段、方法和行为，比字符串错误更有表达力。

[Go by Example 的 Custom Errors](https://gobyexample.com/custom-errors) 用一个参数错误演示了自定义类型和 `errors.As`。本文补充指针接收者、错误链、typed nil 和设计建议。

## 完整示例

```go
package main

import (
	"errors"
	"fmt"
)

type argError struct {
	arg     int
	message string
}

func (e *argError) Error() string {
	return fmt.Sprintf("%d - %s", e.arg, e.message)
}

func f(arg int) (int, error) {
	if arg == 42 {
		return -1, &argError{arg, "can't work with it"}
	}
	return arg + 3, nil
}

func main() {
	_, err := f(42)

	var ae *argError
	if errors.As(err, &ae) {
		fmt.Println(ae.arg)
		fmt.Println(ae.message)
	} else {
		fmt.Println("err doesn't match argError")
	}
}
```

运行：

```bash
$ go run custom-errors.go
42
can't work with it
```

## 定义错误类型

```go
type argError struct {
	arg     int
	message string
}

func (e *argError) Error() string {
	return fmt.Sprintf("%d - %s", e.arg, e.message)
}
```

只要实现了 `Error() string`，`*argError` 就满足 `error` 接口。命名上通常以 `Error` 结尾，例如 `argError`、`ValidationError`、`NotFoundError`。

错误类型可以携带任意字段：

```go
type ValidationError struct {
	Field string
	Value any
	Rule  string
}

func (e *ValidationError) Error() string {
	return fmt.Sprintf("field %s violates rule %s", e.Field, e.Rule)
}
```

调用方可以用 `errors.As` 取出字段，做针对性处理。

## 指针接收者还是值接收者

`Error()` 可以定义在值接收者或指针接收者上：

```go
func (e argError) Error() string  // 值和指针都实现 error
func (e *argError) Error() string // 只有指针实现 error
```

如果错误类型带有可变字段，或希望避免复制，通常使用指针接收者并返回 `&argError{...}`。

使用指针接收者时，只有 `*argError` 满足 `error`，`argError` 不满足。返回错误时不要写成 `argError{...}`：

```go
return -1, argError{42, "bad"} // 编译错误：argError 没有实现 error
```

## 用 `errors.As` 提取错误

```go
var ae *argError
if errors.As(err, &ae) {
	fmt.Println(ae.arg)
	fmt.Println(ae.message)
}
```

`errors.As` 会沿着错误链查找第一个类型为 `*argError` 的错误，并把它赋给 `ae`。

目标参数必须是指针，且指向的类型要实现 `error`：

```go
var ae *argError
errors.As(err, &ae) // 正确
```

不要写成 `errors.As(err, ae)`，那会传入 nil 指针，直接 panic 或返回 false。

## 用 `errors.AsType` 提取

较新的 Go 版本提供了泛型版本：

```go
if ae, ok := errors.AsType[*argError](err); ok {
	fmt.Println(ae.arg)
	fmt.Println(ae.message)
}
```

`AsType` 返回匹配的值和布尔值，不需要预先声明目标变量。它和 `errors.As` 的关系类似泛型函数和传统写法。

如果项目需要兼容旧版本 Go，继续使用 `errors.As`。

## 错误链与 `Unwrap`

自定义错误也可以包装底层错误：

```go
type QueryError struct {
	Query string
	Err   error
}

func (e *QueryError) Error() string {
	return fmt.Sprintf("query %q: %v", e.Query, e.Err)
}

func (e *QueryError) Unwrap() error {
	return e.Err
}
```

实现 `Unwrap() error` 后，`errors.Is` 和 `errors.As` 可以继续向下查找：

```go
err := &QueryError{Query: "select ...", Err: sql.ErrNoRows}
fmt.Println(errors.Is(err, sql.ErrNoRows)) // true
```

如果只想让调用方看到自定义错误，不需要暴露底层错误，可以不实现 `Unwrap`。

## 自定义 `Is` 方法

有时两个错误值不是同一个对象，但逻辑上等价。可以实现 `Is` 方法：

```go
func (e *argError) Is(target error) bool {
	t, ok := target.(*argError)
	return ok && t.arg == e.arg
}
```

之后 `errors.Is(err, &argError{arg: 42})` 会调用这个方法判断。`Is` 适合定义“同一类错误”的语义，不要滥用。

## typed nil 陷阱

一个指向 nil 的指针赋给 `error` 接口后，接口本身不是 nil：

```go
var e *argError
var err error = e

fmt.Println(e == nil)   // true
fmt.Println(err == nil) // false
```

因为 `err` 的动态类型是 `*argError`，动态值是 nil。调用方 `if err != nil` 会进入错误分支，但访问 `e.arg` 会 panic。

正确做法：

```go
if e == nil {
	return nil
}
return e
```

或者在函数中直接返回 `nil`，不要把 nil 指针装进接口。

## 错误类型的设计建议

- 错误类型只暴露调用方真正需要的信息；
- 字段尽量不可变，避免并发修改；
- 错误消息包含上下文，但不要泄露敏感信息；
- 需要链式判断时实现 `Unwrap`；
- 需要自定义匹配语义时实现 `Is`；
- 不要把所有错误都定义成一种大类型。

## 常见错误

### 1. 指针接收者却返回值类型

`argError` 没有实现 `error`，只有 `*argError` 实现。返回 `argError{}` 会编译失败。

### 2. `errors.As` 目标写错

必须传入指向错误类型的指针，例如 `var ae *argError; errors.As(err, &ae)`。不要传 `ae` 本身。

### 3. 忘记 `Unwrap`

包装了底层错误却没有 `Unwrap`，`errors.Is` / `errors.As` 无法继续向下查找。

### 4. 返回 nil 指针作为 error

接口不为 nil，调用方会误判。返回前判断指针是否为 nil。

### 5. 错误类型过于复杂

错误类型应聚焦于调用方需要的信息。不要把整个请求上下文塞进错误对象。

### 6. 用字符串比较判断自定义错误

应使用 `errors.As` 提取类型，或用 `errors.Is` 匹配哨兵错误。

## 和 Java 对照

| 概念 | Go | Java |
| --- | --- | --- |
| 自定义错误 | 实现 `Error() string` 的类型 | 继承 `Exception` / `RuntimeException` |
| 携带字段 | 结构体字段 | 异常字段 |
| 类型判断 | `errors.As` / `errors.AsType` | `instanceof` / `catch` |
| 包装原因 | `Unwrap` / `%w` | `cause` |
| 错误消息 | `Error()` | `getMessage()` |
| 堆栈 | 默认不附带 | 默认附带 |

Java 的自定义异常通过继承表达，Go 通过实现接口表达。Java 异常自动带堆栈，Go 的错误类型默认只有字段和消息，需要额外记录堆栈。

## 动手练习

1. 定义 `ValidationError`，包含字段名、值和规则，实现 `Error()`。
2. 写一个函数，在参数非法时返回 `*ValidationError`。
3. 用 `errors.As` 提取错误并打印字段。
4. 用 `errors.AsType` 改写上面的提取逻辑。
5. 给自定义错误实现 `Unwrap`，包装一个哨兵错误，并用 `errors.Is` 验证。
6. 构造 typed nil 错误，观察 `err != nil` 的行为。

## 小结

- 自定义错误是实现 `Error() string` 的类型，通常以 `Error` 结尾。
- 指针接收者只让指针类型满足 `error`，返回值类型要匹配。
- `errors.As` / `errors.AsType` 用于从错误链中提取具体类型。
- 实现 `Unwrap` 可以保留底层错误，支持链式判断。
- nil 指针装入 error 接口后，接口本身不等于 nil。
- 错误类型应聚焦调用方需要的信息，不要过度设计。

---

上一篇：[[Go/Advanced/Errors|Go 进阶：错误处理]]
下一篇：[[Go/Concurrency/Goroutines|Go 并发：goroutine]]

返回 [[Go/Advanced/index|Go 进阶]]。
