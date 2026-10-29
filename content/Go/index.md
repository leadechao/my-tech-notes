---
title: Go 学习笔记
description: 从 Go by Example 出发，记录 Go 语言的基础语法、并发模型与工程实践。
created: 2026-10-29
modified: 2026-10-29
---

# Go 学习笔记

这里记录我学习 Go 的完整路径。笔记以 [Go by Example](https://gobyexample.com/) 的示例为线索，但不只是翻译示例：每个知识点都会补充运行机制、常见错误，以及和 Java 等语言的对照。

## 学习路线

- **基础**：包、变量、类型、流程控制、集合、错误处理
- **进阶**：函数、指针、字符串、结构体、方法、接口、枚举、嵌入、泛型、迭代器、错误处理
- **并发**：goroutine、channel、`select`、同步原语、并发模式
- **工程化**：模块、测试、基准测试、性能分析、部署

## 文章列表

### 基础

**入门**

- [[Go/Basics/Hello World|Go 入门：从 Hello World 开始]] —— 最小程序结构、`go run` 与 `go build`、常见编译错误

**值与变量**

- [[Go/Basics/Values|Go 基础：值类型与字面量]] —— 基本类型、默认类型、整数除法、浮点精度与类型转换
- [[Go/Basics/Variables|Go 基础：变量声明与零值]] —— `var`、`:=`、零值、作用域、变量遮蔽与包级变量
- [[Go/Basics/Constants|Go 基础：常量、无类型常量与 iota]] —— 编译期常量、高精度表达式、typed/untyped 与枚举

**流程控制**

- [[Go/Basics/For|Go 基础：for 循环与 range]] —— 条件循环、三段式、`range`、`break`/`continue` 与标签
- [[Go/Basics/If-Else|Go 基础：if/else 条件分支]] —— 条件写法、初始化语句、作用域、`else` 位置与无三元运算符
- [[Go/Basics/Switch|Go 基础：switch 与类型分支]] —— 表达式 switch、多值 case、无表达式 switch、`fallthrough` 与类型 switch

**集合**

- [[Go/Basics/Arrays|Go 基础：数组]] —— 固定长度、零值、字面量、多维数组与值语义
- [[Go/Basics/Slices|Go 基础：切片]] —— nil 切片、`make`、`append`、`copy`、切片表达式与底层数组
- [[Go/Basics/Maps|Go 基础：map]] —— 创建、读写、comma-ok、`delete`/`clear`、遍历顺序与 nil map

### 进阶

**函数**

- [[Go/Advanced/Functions|Go 进阶：函数]] —— 声明、参数简写、显式返回、一等函数与值传递
- [[Go/Advanced/Multiple Return Values|Go 进阶：多返回值]] —— 多返回值、空白标识符、命名返回值与 result/error
- [[Go/Advanced/Variadic Functions|Go 进阶：变参函数]] —— `...T`、切片展开、定参混合与底层共享
- [[Go/Advanced/Closures|Go 进阶：闭包]] —— 匿名函数、变量捕获、独立状态与循环变量
- [[Go/Advanced/Recursion|Go 进阶：递归]] —— 基准条件、匿名递归、调用栈与性能陷阱
- [[Go/Advanced/Range Over Built-in Types|Go 进阶：range 遍历内置类型]] —— 数组、切片、map、字符串、channel 与整数

**指针与字符串**

- [[Go/Advanced/Pointers|Go 进阶：指针]] —— `&`、`*`、nil 指针、`new`、结构体指针与使用边界
- [[Go/Advanced/Strings and Runes|Go 进阶：字符串与 rune]] —— UTF-8 字节、rune、range 解码、字符数与截取

**类型与抽象**

- [[Go/Advanced/Structs|Go 进阶：结构体]] —— 字段、字面量、零值、值语义、比较与匿名结构体
- [[Go/Advanced/Methods|Go 进阶：方法]] —— 值/指针接收者、方法集、接口实现与 nil 接收者
- [[Go/Advanced/Interfaces|Go 进阶：接口]] —— 隐式实现、多态、类型断言、空接口与 typed nil
- [[Go/Advanced/Enums|Go 进阶：枚举]] —— 命名类型、`iota`、Stringer、非法值与 JSON
- [[Go/Advanced/Struct Embedding|Go 进阶：结构体嵌入]] —— 字段/方法提升、接口复用、遮蔽与歧义

**泛型**

- [[Go/Advanced/Generics|Go 进阶：泛型]] —— 类型参数、约束、`comparable`、`~`、泛型类型与类型推断

**迭代器与错误**

- [[Go/Advanced/Range Over Iterators|Go 进阶：range 遍历迭代器]] —— `iter.Seq`、`yield`、惰性迭代、无限序列与标准库工具
- [[Go/Advanced/Errors|Go 进阶：错误处理]] —— `error`、哨兵错误、`%w`、`errors.Is` / `As` 与错误链
- [[Go/Advanced/Custom Errors|Go 进阶：自定义错误]] —— 自定义错误类型、`errors.AsType`、`Unwrap` 与 typed nil

### 并发

**并发基础**

- [[Go/Concurrency/Goroutines|Go 并发：goroutine]] —— `go` 关键字、调度、主 goroutine 退出、循环变量与数据竞争
- [[Go/Concurrency/Channels|Go 并发：channel]] —— 发送接收、阻塞、缓冲、关闭、方向、nil channel 与死锁
- [[Go/Concurrency/Range over Channels|Go 并发：range 遍历 channel]] —— 关闭语义、剩余值、`ok` 判断与发送方关闭规则

**时间控制**

- [[Go/Concurrency/Timers|Go 并发：定时器 Timer]] —— `NewTimer`、`Stop`、`time.After` 与超时控制
- [[Go/Concurrency/Tickers|Go 并发：周期定时器 Ticker]] —— 周期触发、`Stop`、丢 tick 与 `time.Tick` 泄漏

**并发模式**

- [[Go/Concurrency/Worker Pools|Go 并发：工作池]] —— jobs/results channel、固定 worker、关闭与结果收集
- [[Go/Concurrency/Rate Limiting|Go 并发：限流]] —— 固定速率、突发令牌桶、`x/time/rate` 与生产实践

**同步与共享**

- [[Go/Concurrency/WaitGroups|Go 并发：WaitGroup]] —— `Add`/`Done`/`Wait`、`wg.Go`、复制禁忌与错误传播
- [[Go/Concurrency/Atomic Counters|Go 并发：原子计数器]] —— `sync/atomic`、`Uint64`、CAS 与原子操作边界
- [[Go/Concurrency/Mutexes|Go 并发：互斥锁]] —— `sync.Mutex`、锁粒度、`RWMutex`、死锁与复制禁忌

## 推荐资源

- [Go by Example](https://gobyexample.com/)：以可运行示例驱动的语法速查
- [A Tour of Go](https://go.dev/tour/welcome/1)：官方交互式入门教程
- [Effective Go](https://go.dev/doc/effective_go)：写出地道 Go 代码的必读文档
- [The Go Programming Language Specification](https://go.dev/ref/spec)：遇到边界问题时查阅的权威规范
