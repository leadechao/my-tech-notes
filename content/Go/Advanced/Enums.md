---
title: "Go 进阶：枚举"
description: "从 Go by Example 的 Enums 示例出发，用命名类型、const、iota 和 Stringer 实现类型安全的枚举，并理解它的边界。"
tags:
  - Go
  - 编程语言
  - 进阶
  - 枚举
created: 2026-10-29
modified: 2026-10-29
---

# Go 进阶：枚举

Go 没有 `enum` 关键字，也没有 Java 那样的枚举类。枚举是用“命名类型 + 常量 + `iota`”模拟出来的约定。

这种实现方式很轻量，但不是封闭集合：你可以把任意整数转换成枚举类型。理解这一点，才能写出既类型安全又能处理非法值的代码。

[Go by Example 的 Enums](https://gobyexample.com/enums) 用服务器状态演示了这套惯用法。本文补充类型安全、Stringer、非法值和工程实践。

## 完整示例

```go
package main

import "fmt"

type ServerState int

const (
	StateIdle ServerState = iota
	StateConnected
	StateError
	StateRetrying
)

var stateName = map[ServerState]string{
	StateIdle:      "idle",
	StateConnected: "connected",
	StateError:     "error",
	StateRetrying:  "retrying",
}

func (ss ServerState) String() string {
	return stateName[ss]
}

func transition(s ServerState) ServerState {
	switch s {
	case StateIdle:
		return StateConnected
	case StateConnected, StateRetrying:
		return StateIdle
	case StateError:
		return StateError
	default:
		panic(fmt.Errorf("unknown state: %s", s))
	}
}

func main() {
	ns := transition(StateIdle)
	fmt.Println(ns)

	ns2 := transition(ns)
	fmt.Println(ns2)
}
```

运行：

```bash
$ go run enums.go
connected
idle
```

## 命名类型 + 常量 + iota

```go
type ServerState int

const (
	StateIdle ServerState = iota
	StateConnected
	StateError
	StateRetrying
)
```

三部分：

- `type ServerState int` 定义一个新的命名类型，底层是 `int`；
- `const` 定义枚举值；
- `iota` 从 0 开始，每个常量规格递增一次。

最终得到：

```text
StateIdle      = 0
StateConnected = 1
StateError     = 2
StateRetrying  = 3
```

也可以显式指定值，或跳过某些值：

```go
const (
	StateUnknown ServerState = iota // 0
	StateIdle                       // 1
	StateConnected                  // 2
)
```

`iota` 的细节见常量一篇。

## 类型安全到什么程度

枚举类型能提供一定的编译期检查：

```go
var state int = 1
transition(state)
// cannot use state (variable of type int) as ServerState value
```

一个有类型的 `int` 变量不能直接传给 `ServerState` 参数，必须显式转换。

但无类型常量是例外：

```go
transition(1) // 可以编译，1 被当作 StateConnected
```

因为 `1` 是无类型常量，可以隐式转换成 `ServerState`。所以枚举并不是完全封闭的，调用方仍然可以传数字字面量。

更严重的是，显式转换总是允许的：

```go
s := ServerState(99) // 编译通过
```

`99` 不在枚举定义中，但类型系统不会阻止你。需要在业务逻辑中校验。

## Stringer：让枚举打印成名字

实现 `fmt.Stringer` 接口后，`fmt` 打印时会调用 `String()`：

```go
func (ss ServerState) String() string {
	return stateName[ss]
}
```

```go
fmt.Println(StateConnected) // connected
```

`fmt.Stringer` 的定义是：

```go
type Stringer interface {
	String() string
}
```

这是一个只有一个方法的典型小接口。

## 用 map 维护名称

```go
var stateName = map[ServerState]string{
	StateIdle:      "idle",
	StateConnected: "connected",
	StateError:     "error",
	StateRetrying:  "retrying",
}
```

新增枚举值时，必须同步更新这个 map，否则 `String()` 会返回空字符串。也可以改用 `switch`：

```go
func (ss ServerState) String() string {
	switch ss {
	case StateIdle:
		return "idle"
	case StateConnected:
		return "connected"
	case StateError:
		return "error"
	case StateRetrying:
		return "retrying"
	default:
		return "unknown"
	}
}
```

`switch` 写法更冗长，但新增值时编译器不会提醒你补全所有分支（Go 没有枚举穷尽性检查），仍需人工维护。

值很多时，可以使用 `stringer` 工具自动生成 `String()`：

```go
//go:generate stringer -type=ServerState
```

然后执行 `go generate`。这是标准库和大型项目中常见的做法。

## 状态转换与非法值

```go
func transition(s ServerState) ServerState {
	switch s {
	case StateIdle:
		return StateConnected
	case StateConnected, StateRetrying:
		return StateIdle
	case StateError:
		return StateError
	default:
		panic(fmt.Errorf("unknown state: %s", s))
	}
}
```

`default` 分支处理非法值：可能是 `ServerState(99)`，也可能是未来新增但忘记处理的状态。生产代码中应根据场景选择 panic、返回错误或回退到默认状态。

注意：如果 `String()` 对非法值返回空字符串，`panic` 信息可能变成 `unknown state: `，排查时不够直观。可以在 `String()` 的 default 分支返回 `fmt.Sprintf("ServerState(%d)", int(ss))`。

## 零值问题

枚举的零值通常落在第一个常量上：

```go
var s ServerState // 0，即 StateIdle
```

这很方便，但也可能掩盖“没有初始化”的问题。如果零值不应该是合法状态，可以把 `StateUnknown` 放在第一位，或在使用前显式校验。

## 枚举与 JSON

`String()` 只影响 `fmt` 的打印，不会自动影响 JSON 编码。枚举默认会被编码成底层整数：

```go
json.Marshal(StateConnected) // 1
```

如果需要编码成字符串，应实现 `MarshalJSON` / `UnmarshalJSON`，或使用 `encoding.TextMarshaler`。不要以为实现了 `String()` 就自动拥有了字符串序列化。

## 常见错误

### 1. 以为枚举是封闭集合

`ServerState(99)` 永远可以编译。需要在入口处校验，或在 `default` 分支处理。

### 2. 用裸 `int` 代替枚举类型

```go
func transition(s int)
```

这样会失去类型检查。应使用 `ServerState`。

### 3. 新增常量后忘记更新 `String()` 或 `switch`

Go 不会检查枚举是否穷尽，新增值可能悄悄落入 `default`。

### 4. `String()` 对未知值返回空字符串

排查问题时信息不足。建议返回 `ServerState(99)` 这样的可读表示。

### 5. 认为零值一定是“无效”

零值通常是第一个枚举值。如果不希望零值合法，显式定义 `Unknown` 并放在第一位。

### 6. 以为 `String()` 影响 JSON

JSON 编码使用自己的规则。需要字符串枚举时实现 `MarshalJSON` 或 `TextMarshaler`。

## 和 Java 对照

| 概念 | Go | Java |
| --- | --- | --- |
| 语言特性 | 无 enum 关键字 | 原生 `enum` |
| 类型安全 | 命名类型 + 常量，非封闭 | 封闭集合，类型安全 |
| 底层值 | 通常 `int` + `iota` | `ordinal()` |
| 名称 | 实现 `String()` | `name()` / `toString()` |
| 遍历所有值 | 手动维护切片/map | `values()` |
| 附加数据 | 用 map 或方法 | 枚举字段/构造器 |
| 非法值 | 可通过转换构造 | 不能构造未定义值 |

Java 的枚举是真正的类，可以有字段、构造器和方法，取值集合固定；Go 的枚举只是命名常量的约定，更轻量，但没有封闭性保证。

## 动手练习

1. 定义 `Weekday` 类型和七个常量，用 `iota` 赋值。
2. 实现 `String()`，让 `fmt.Println` 输出星期名称。
3. 写一个 `Next` 方法，返回下一天。
4. 构造 `Weekday(99)`，观察 `String()` 和业务函数的行为。
5. 用 `switch` 实现 `String()`，对比 map 写法。
6. 给枚举实现 `MarshalJSON`，让 JSON 输出字符串而不是数字。

## 小结

- Go 用命名类型 + `const` + `iota` 模拟枚举。
- 命名类型提供一定的类型检查，但枚举不是封闭集合。
- 实现 `String()` 可以让 `fmt` 输出可读名称。
- 新增枚举值时要同步更新名称映射和分支逻辑。
- 零值通常是第一个枚举值，需要根据业务决定是否合法。
- `String()` 不影响 JSON，序列化需要单独实现。

---

上一篇：[[Go/Advanced/Interfaces|Go 进阶：接口]]
下一篇：[[Go/Advanced/Struct Embedding|Go 进阶：结构体嵌入]]

返回 [[Go/Advanced/index|Go 进阶]]。
