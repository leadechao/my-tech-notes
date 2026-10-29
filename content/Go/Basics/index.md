---
title: 基础
description: Go 基础语法学习笔记：从最小可运行程序开始，逐步掌握变量、类型、流程控制与函数。
created: 2026-10-29
modified: 2026-10-29
---

# Go 基础

这一部分从最小可运行程序开始，逐步补齐 Go 的基础语法。每篇笔记都对应一个可以独立运行的示例，建议边读边在本机敲一遍。

学习时不必急着记住所有语法，先关注三件事：**代码如何组织、程序如何运行、错误如何暴露**。Go 的编译器在这些方面非常直接，习惯之后会省下大量排查时间。

## 文章

### 入门

- [[Go/Basics/Hello World|Go 入门：从 Hello World 开始]] —— 包声明、导入、入口函数，以及 `go run` / `go build` 的区别

### 值与变量

- [[Go/Basics/Values|Go 基础：值类型与字面量]] —— 字符串、整数、浮点数、布尔值、默认类型与显式转换
- [[Go/Basics/Variables|Go 基础：变量声明与零值]] —— `var`、`:=`、多变量赋值、零值、作用域与变量遮蔽
- [[Go/Basics/Constants|Go 基础：常量、无类型常量与 iota]] —— `const`、高精度常量表达式、类型化常量与枚举

### 流程控制

- [[Go/Basics/For|Go 基础：for 循环与 range]] —— 条件循环、三段式、`range`、`break`/`continue` 与标签
- [[Go/Basics/If-Else|Go 基础：if/else 条件分支]] —— 条件写法、初始化语句、作用域、`else` 位置与无三元运算符
- [[Go/Basics/Switch|Go 基础：switch 与类型分支]] —— 表达式 switch、多值 case、无表达式 switch、`fallthrough` 与类型 switch

### 集合

- [[Go/Basics/Arrays|Go 基础：数组]] —— 固定长度、零值、字面量、多维数组与值语义
- [[Go/Basics/Slices|Go 基础：切片]] —— nil 切片、`make`、`append`、`copy`、切片表达式与底层数组
- [[Go/Basics/Maps|Go 基础：map]] —— 创建、读写、comma-ok、`delete`/`clear`、遍历顺序与 nil map

## 学习建议

1. 每个示例都亲手运行，不要只读代码。
2. 遇到报错先读完整信息，Go 的错误提示通常已经指出了文件和行号。
3. 坚持用 `gofmt` 格式化，尽早适应官方代码风格。
4. 学完一个知识点后，尝试改动示例并预测输出，再运行验证。

---

返回 [[Go/index|Go 学习笔记]]。
