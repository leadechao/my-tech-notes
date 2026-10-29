---
title: 进阶
description: Go 进阶学习笔记：函数、指针、字符串、结构体、方法、接口、枚举、嵌入、泛型、迭代器与错误处理。
created: 2026-10-29
modified: 2026-10-29
---

# Go 进阶

基础篇解决了“能写、能跑”的问题；进阶篇关注如何用 Go 的方式组织行为、数据和抽象。

这一部分从函数开始，经过指针、字符串、结构体、方法、接口，最后到达枚举、嵌入和泛型。每篇都尽量给出可运行示例、常见错误和 Java 对照。

## 文章

### 函数

- [[Go/Advanced/Functions|Go 进阶：函数]] —— 声明、参数简写、显式返回、一等函数与值传递
- [[Go/Advanced/Multiple Return Values|Go 进阶：多返回值]] —— 多返回值、空白标识符、命名返回值与 result/error
- [[Go/Advanced/Variadic Functions|Go 进阶：变参函数]] —— `...T`、切片展开、定参混合与底层共享
- [[Go/Advanced/Closures|Go 进阶：闭包]] —— 匿名函数、变量捕获、独立状态与循环变量
- [[Go/Advanced/Recursion|Go 进阶：递归]] —— 基准条件、匿名递归、调用栈与性能陷阱
- [[Go/Advanced/Range Over Built-in Types|Go 进阶：range 遍历内置类型]] —— 数组、切片、map、字符串、channel 与整数

### 指针与字符串

- [[Go/Advanced/Pointers|Go 进阶：指针]] —— `&`、`*`、nil 指针、`new`、结构体指针与使用边界
- [[Go/Advanced/Strings and Runes|Go 进阶：字符串与 rune]] —— UTF-8 字节、rune、range 解码、字符数与截取

### 类型与抽象

- [[Go/Advanced/Structs|Go 进阶：结构体]] —— 字段、字面量、零值、值语义、比较与匿名结构体
- [[Go/Advanced/Methods|Go 进阶：方法]] —— 值/指针接收者、方法集、接口实现与 nil 接收者
- [[Go/Advanced/Interfaces|Go 进阶：接口]] —— 隐式实现、多态、类型断言、空接口与 typed nil
- [[Go/Advanced/Enums|Go 进阶：枚举]] —— 命名类型、`iota`、Stringer、非法值与 JSON
- [[Go/Advanced/Struct Embedding|Go 进阶：结构体嵌入]] —— 字段/方法提升、接口复用、遮蔽与歧义

### 泛型

- [[Go/Advanced/Generics|Go 进阶：泛型]] —— 类型参数、约束、`comparable`、`~`、泛型类型与类型推断

### 迭代器与错误

- [[Go/Advanced/Range Over Iterators|Go 进阶：range 遍历迭代器]] —— `iter.Seq`、`yield`、惰性迭代、无限序列与标准库工具
- [[Go/Advanced/Errors|Go 进阶：错误处理]] —— `error`、哨兵错误、`%w`、`errors.Is` / `As` 与错误链
- [[Go/Advanced/Custom Errors|Go 进阶：自定义错误]] —— 自定义错误类型、`errors.AsType`、`Unwrap` 与 typed nil

## 学习建议

1. 先确保基础篇的变量、切片、map 已经熟练，再进入这一部分。
2. 函数、方法、接口是一条主线，建议按顺序阅读。
3. 遇到指针和值语义问题时，画一下内存和复制关系。
4. 泛型不要为了用而用，先写具体类型，发现重复后再抽象。
5. 每篇的 Java 对照可以帮助迁移已有经验，但不要假设两种语言的模型完全一样。

---

上一阶段：[[Go/Basics/index|Go 基础]]

下一阶段：[[Go/Concurrency/index|Go 并发]]

返回 [[Go/index|Go 学习笔记]]。
