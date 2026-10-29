---
title: Go 学习笔记
description: 从 Go by Example 出发，记录 Go 语言的基础语法、并发模型与工程实践。
created: 2026-10-29
modified: 2026-10-29
---

# Go 学习笔记

这里记录我学习 Go 的完整路径。笔记以 [Go by Example](https://gobyexample.com/) 的示例为线索，但不只是翻译示例：每个知识点都会补充运行机制、常见错误，以及和 Java 等语言的对照。

## 学习路线

- **基础**：包、变量、函数、流程控制、集合、错误处理
- **进阶**：结构体、接口、泛型、反射
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

## 推荐资源

- [Go by Example](https://gobyexample.com/)：以可运行示例驱动的语法速查
- [A Tour of Go](https://go.dev/tour/welcome/1)：官方交互式入门教程
- [Effective Go](https://go.dev/doc/effective_go)：写出地道 Go 代码的必读文档
- [The Go Programming Language Specification](https://go.dev/ref/spec)：遇到边界问题时查阅的权威规范
