---
title: "Go 进阶：错误处理"
description: "从 Go by Example 的 Errors 示例出发，理解 error 接口、errors.New、哨兵错误、%w 包装、errors.Is/As 与错误链。"
tags:
  - Go
  - 编程语言
  - 进阶
  - 错误处理
created: 2026-10-29
modified: 2026-10-29
---

# Go 进阶：错误处理

Go 把错误当作普通返回值，而不是异常。函数显式返回 `error`，调用方显式检查。这样代码的控制流一目了然，但也要求开发者认真处理每一个错误。

[Go by Example 的 Errors](https://gobyexample.com/errors) 演示了错误返回、哨兵错误、包装和 `errors.Is`。本文补充 `errors.As`、`errors.Join`、错误字符串约定和常见误区。

## 完整示例

```go
package main

import (
	"errors"
	"fmt"
)

func f(arg int) (int, error) {
	if arg == 42 {
		return -1, errors.New("can't work with 42")
	}
	return arg + 3, nil
}

var ErrOutOfTea = errors.New("no more tea available")
var ErrPower = errors.New("can't boil water")

func makeTea(arg int) error {
	if arg == 2 {
		return ErrOutOfTea
	} else if arg == 4 {
		return fmt.Errorf("making tea: %w", ErrPower)
	}
	return nil
}

func main() {
	for _, i := range []int{7, 42} {
		if r, e := f(i); e != nil {
			fmt.Println("f failed:", e)
		} else {
			fmt.Println("f worked:", r)
		}
	}

	for i := range 5 {
		if err := makeTea(i); err != nil {
			if errors.Is(err, ErrOutOfTea) {
				fmt.Println("We should buy new tea!")
			} else if errors.Is(err, ErrPower) {
				fmt.Println("Now it is dark.")
			} else {
				fmt.Printf("unknown error: %s\n", err)
			}
			continue
		}
		fmt.Println("Tea is ready!")
	}
}
```

运行：

```bash
$ go run errors.go
f worked: 10
f failed: can't work with 42
Tea is ready!
Tea is ready!
We should buy new tea!
Tea is ready!
Now it is dark.
```

## `error` 是一个接口

```go
type error interface {
	Error() string
}
```

任何实现了 `Error() string` 的类型都可以作为错误。标准库的 `errors.New` 返回一个简单的错误值，`fmt.Errorf` 返回格式化后的错误。

错误通常作为最后一个返回值：

```go
func f(arg int) (int, error) {
	if arg == 42 {
		return -1, errors.New("can't work with 42")
	}
	return arg + 3, nil
}
```

`nil` 表示没有错误。调用方必须显式检查：

```go
r, err := f(7)
if err != nil {
	fmt.Println("failed:", err)
	return
}
fmt.Println("result:", r)
```

`r` 和 `err` 可以同时有值，但按照惯例，出错时结果应为零值，且调用方不应使用它。

## 哨兵错误

哨兵错误（sentinel error）是包级别预定义的错误值，用于表示特定条件：

```go
var ErrOutOfTea = errors.New("no more tea available")
var ErrPower = errors.New("can't boil water")
```

调用方可以用 `errors.Is` 判断：

```go
if errors.Is(err, ErrOutOfTea) {
	fmt.Println("buy more tea")
}
```

哨兵错误适合调用方需要区分的、稳定的错误条件。错误消息可以变化，但错误值本身应保持稳定。

## 包装错误

直接返回底层错误会丢失上下文。用 `fmt.Errorf` 的 `%w` 动词包装：

```go
func makeTea(arg int) error {
	if arg == 2 {
		return ErrOutOfTea
	} else if arg == 4 {
		return fmt.Errorf("making tea: %w", ErrPower)
	}
	return nil
}
```

`%w` 会把原错误嵌入新错误，形成错误链：

```text
making tea: can't boil water
  └── can't boil water
```

`%v` 只会把错误转成字符串，丢失错误链：

```go
fmt.Errorf("making tea: %v", ErrPower) // 不能再用 errors.Is 匹配 ErrPower
```

需要调用方用 `errors.Is` / `errors.As` 检查时，必须使用 `%w`。

## `errors.Is`：判断错误链中是否有目标

```go
if errors.Is(err, ErrOutOfTea) {
	// err 本身是 ErrOutOfTea，或它的链中包含 ErrOutOfTea
}
```

`errors.Is` 会沿着 `Unwrap()` 链逐层比较。它比 `err == ErrOutOfTea` 更可靠，因为后者只能比较最外层错误。

`errors.Is` 也可以比较实现了 `Is(error) bool` 的自定义错误。

## `errors.As`：提取特定类型的错误

`errors.Is` 判断“是不是某个错误值”，`errors.As` 则用于“是不是某个错误类型”，并把错误赋给目标变量：

```go
var pathErr *os.PathError
if errors.As(err, &pathErr) {
	fmt.Println("path:", pathErr.Path)
}
```

目标必须是指向错误类型的指针。`errors.As` 会沿着错误链查找第一个匹配的类型。

Go 较新版本提供了泛型版本 `errors.AsType`：

```go
if pathErr, ok := errors.AsType[*os.PathError](err); ok {
	fmt.Println("path:", pathErr.Path)
}
```

它返回匹配的值和布尔值，不需要预先声明目标变量。旧代码和旧版本 Go 仍以 `errors.As` 为主。

## `errors.Join`：组合多个错误

Go 1.20 引入 `errors.Join`，可以把多个错误合并成一个：

```go
err := errors.Join(err1, err2)
```

合并后的错误实现 `Unwrap() []error`，`errors.Is` / `errors.As` 会遍历所有子错误。适合“批量操作中多个步骤失败”的场景。

## 错误字符串的约定

Go 社区有一些约定：

- 错误消息通常小写开头，不加句号；
- 错误消息描述发生了什么，而不是“出错了”；
- 不要在错误字符串里重复包名或函数名；
- 不要通过比较错误字符串来判断错误类型，使用 `errors.Is` / `errors.As`。

例如：

```go
return fmt.Errorf("open config: %w", err)
```

比 `return errors.New("Error: failed to open config file")` 更有用，因为它保留了底层原因和上下文。

## panic 与 recover

`panic` 用于不可恢复的程序错误，例如数组越界、nil 解引用、违反不变量。普通业务错误应返回 `error`，不要用 `panic` 控制流程。

`recover` 只能在 `defer` 中生效，用于捕获 panic：

```go
defer func() {
	if r := recover(); r != nil {
		fmt.Println("recovered:", r)
	}
}()
```

在 HTTP 服务等长期运行的进程中，顶层 recover 可以防止单个请求导致整个进程崩溃，但不能替代正常的错误处理。

## 常见错误

### 1. 忽略错误

```go
r, _ := f(42)
```

忽略了错误，调用方无法知道失败原因。除非有明确理由，否则应检查 `err`。

### 2. 用 `%v` 包装却想用 `errors.Is`

`%v` 只保留字符串，不保留错误链。需要链式判断时使用 `%w`。

### 3. 直接比较错误字符串

错误消息可能变化，字符串比较不可靠。使用哨兵错误、`errors.Is` 或自定义错误类型。

### 4. 用 `==` 比较包装后的错误

`err == ErrPower` 在错误被包装后为 false。使用 `errors.Is`。

### 5. 把 nil 指针装入 error 接口

如果 `*MyError` 实现了 `error`，返回 nil 指针会得到非 nil 的 error 接口。应在确认非 nil 时再返回，或直接返回 `nil`。

### 6. 滥用 panic

普通业务错误应返回 `error`。`panic` 只用于程序无法继续执行的场景。

## 和 Java 对照

| 概念 | Go | Java |
| --- | --- | --- |
| 错误机制 | 显式返回值 | 异常 `throw/catch` |
| 错误类型 | `error` 接口 | `Throwable` 子类 |
| 检查时机 | 编译期不强制检查 | 受检异常编译期强制 |
| 包装 | `%w` + `Unwrap` | `cause` + `initCause` |
| 类型判断 | `errors.Is` / `errors.As` | `instanceof` / `catch` |
| 堆栈 | 默认不附带 | 默认附带 |
| 恢复 | `recover`（仅 defer） | `catch` |

Java 的异常会自动传播，调用方可以集中捕获；Go 的错误必须逐层返回和检查。Java 异常默认带堆栈，Go 的 `errors.New` 不保存堆栈，需要额外库或 `%+v` 格式化。

## 动手练习

1. 写一个返回 `(int, error)` 的函数，处理参数为 42 的错误。
2. 定义两个哨兵错误，用 `errors.Is` 区分。
3. 用 `%w` 包装底层错误，再用 `errors.Is` 验证链式匹配。
4. 用 `errors.As` 提取 `*os.PathError`。
5. 用 `errors.Join` 合并两个错误。
6. 写一个 `defer + recover`，捕获并打印 panic。

## 小结

- Go 用显式返回的 `error` 表示错误，`nil` 表示成功。
- 哨兵错误用于稳定的错误条件，配合 `errors.Is` 判断。
- `%w` 包装错误并保留错误链，`%v` 只保留字符串。
- `errors.As` 用于提取错误类型，`errors.Is` 用于匹配错误值。
- `errors.Join` 可以组合多个错误。
- 普通业务错误不要用 panic，`recover` 只适合兜底。

---

上一篇：[[Go/Advanced/Range Over Iterators|Go 进阶：range 遍历迭代器]]
下一篇：[[Go/Advanced/Custom Errors|Go 进阶：自定义错误]]

返回 [[Go/Advanced/index|Go 进阶]]。
