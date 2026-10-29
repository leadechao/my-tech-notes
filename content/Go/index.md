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

- [[Go/Basics/Hello World|Go 入门：从 Hello World 开始]] —— 最小程序结构、`go run` 与 `go build`、常见编译错误
- [[Go/Basics/Values|Go 基础：值类型与字面量]] —— 基本类型、默认类型、整数除法、浮点精度与类型转换
- [[Go/Basics/Variables|Go 基础：变量声明与零值]] —— `var`、`:=`、零值、作用域、变量遮蔽与包级变量
- [[Go/Basics/Constants|Go 基础：常量、无类型常量与 iota]] —— 编译期常量、高精度表达式、typed/untyped 与枚举

## 推荐资源

- [Go by Example](https://gobyexample.com/)：以可运行示例驱动的语法速查
- [A Tour of Go](https://go.dev/tour/welcome/1)：官方交互式入门教程
- [Effective Go](https://go.dev/doc/effective_go)：写出地道 Go 代码的必读文档
- [The Go Programming Language Specification](https://go.dev/ref/spec)：遇到边界问题时查阅的权威规范
