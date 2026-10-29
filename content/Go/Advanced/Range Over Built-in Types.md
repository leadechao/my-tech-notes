---
title: "Go 进阶：range 遍历内置类型"
description: "从 Go by Example 的 Range over Built-in Types 示例出发，掌握数组、切片、map、字符串的 range 语义，以及索引、值副本和遍历顺序。"
tags:
  - Go
  - 编程语言
  - 进阶
  - 遍历
created: 2026-10-29
modified: 2026-10-29
---

# Go 进阶：range 遍历内置类型

`range` 是 Go 的通用遍历语法，可以用于数组、切片、map、字符串、channel，以及 Go 1.22 之后的整数。

不同数据结构的 `range` 返回值不同：切片给索引和值，map 给键和值，字符串给字节索引和 rune。理解这些差异，才能避免“为什么拿到的不是我想的那个值”。

[Go by Example 的 Range over Built-in Types](https://gobyexample.com/range-over-built-in-types) 覆盖了切片、map 和字符串。本文补充值副本、遍历顺序、channel 和整数遍历。

## 完整示例

```go
package main

import "fmt"

func main() {
	nums := []int{2, 3, 4}
	sum := 0
	for _, num := range nums {
		sum += num
	}
	fmt.Println("sum:", sum)

	for i, num := range nums {
		if num == 3 {
			fmt.Println("index:", i)
		}
	}

	kvs := map[string]string{"a": "apple", "b": "banana"}
	for k, v := range kvs {
		fmt.Printf("%s -> %s\n", k, v)
	}

	for k := range kvs {
		fmt.Println("key:", k)
	}

	for i, c := range "go" {
		fmt.Println(i, c)
	}
}
```

运行（map 的顺序可能不同）：

```bash
$ go run range-over-built-in-types.go
sum: 9
index: 1
a -> apple
b -> banana
key: a
key: b
0 103
1 111
```

## 数组与切片：索引和值

```go
for i, v := range nums {
	fmt.Println(i, v)
}
```

- 第一个返回值是索引；
- 第二个返回值是元素的值副本。

值为副本这一点很重要：

```go
nums := []int{1, 2, 3}
for _, v := range nums {
	v *= 2
}
fmt.Println(nums) // [1 2 3]
```

修改 `v` 不会修改切片。要修改元素，使用索引：

```go
for i := range nums {
	nums[i] *= 2
}
fmt.Println(nums) // [2 4 6]
```

如果只需要索引，省略第二个变量：

```go
for i := range nums {
	fmt.Println(i)
}
```

如果只需要值，用 `_` 忽略索引：

```go
for _, v := range nums {
	fmt.Println(v)
}
```

## map：键和值

```go
for k, v := range kvs {
	fmt.Printf("%s -> %s\n", k, v)
}
```

- 第一个返回值是键；
- 第二个返回值是值；
- 只写一个变量时，得到的是键。

```go
for k := range kvs {
	fmt.Println(k)
}
```

map 的遍历顺序是随机的，不能依赖。`fmt.Println` 打印 map 时通常会排序，但那是格式化输出的行为，不是 `range` 的保证。

需要在遍历时保持顺序，应先把键取出来排序：

```go
keys := make([]string, 0, len(kvs))
for k := range kvs {
	keys = append(keys, k)
}
slices.Sort(keys)
```

## 字符串：字节索引和 rune

```go
for i, c := range "go" {
	fmt.Println(i, c)
}
// 0 103
// 1 111
```

- 第一个返回值是当前 rune 的起始**字节索引**；
- 第二个返回值是 `rune`，即 Unicode 码点。

`fmt.Println` 打印 `c` 得到的是整数码点（`g` 是 103，`o` 是 111），不是字符。想打印字符，需要 `string(c)` 或 `%c`：

```go
for i, c := range "go" {
	fmt.Printf("%d %c\n", i, c)
}
```

对于多字节字符，索引会跳跃：

```go
for i, c := range "你好" {
	fmt.Printf("%d %c\n", i, c)
}
// 0 你
// 3 好
```

因为每个汉字占 3 个 UTF-8 字节。字符串的 `len` 返回字节数，不是字符数；详细内容见字符串与 rune 一篇。

## channel 和整数

`range` 还可以遍历 channel：

```go
for v := range ch {
	fmt.Println(v)
}
```

它会一直读取，直到 channel 被关闭。如果 channel 永远不关闭，循环也不会结束。

Go 1.22 起，可以遍历整数：

```go
for i := range 3 {
	fmt.Println(i) // 0, 1, 2
}
```

这等价于 `0` 到 `n-1`。旧版本需要使用经典三段式循环。

## nil 切片和 nil map

对 nil 切片和 nil map 使用 `range` 是安全的，循环体不会执行：

```go
var s []int
for range s {
	fmt.Println("never")
}

var m map[string]int
for range m {
	fmt.Println("never")
}
```

这一点和 nil map 写入不同：遍历 nil map 不会 panic。

## 常见错误

### 1. 依赖 map 的遍历顺序

map 的 `range` 顺序随机。需要有序结果时先排序键。

### 2. 修改 range 的值副本

```go
for _, v := range nums {
	v = 0 // 不会修改 nums
}
```

要修改元素，使用索引 `nums[i]`。

### 3. 把字符串索引当成字符位置

字符串的索引是字节偏移。多字节字符之间的索引不连续。

### 4. 只写一个变量却以为拿到值

对数组、切片和 map，只写一个变量拿到的是索引或键，不是值。值需要用第二个变量接收。

### 5. 在旧版本 Go 中遍历整数

`for i := range 3` 需要 Go 1.22+。旧版本会报 `cannot range over 3`。

### 6. 遍历未关闭的 channel

`for v := range ch` 会阻塞到 channel 关闭。确保有发送方关闭 channel，或用 `select` 加退出条件。

### 7. 在遍历 map 时增删键

遍历时删除当前键是允许的，但新增键的行为不确定，可能遍历到也可能遍历不到。需要确定性时，先收集键再处理。

## 和 Java 对照

| 概念 | Go | Java |
| --- | --- | --- |
| 数组/切片 | `for i, v := range xs` | `for (var v : xs)` / 索引循环 |
| map | `for k, v := range m` | `for (var e : m.entrySet())` |
| 只要键 | `for k := range m` | `m.keySet()` |
| 字符串 | 字节索引 + rune | `charAt` / `codePoints()` |
| 顺序 | 切片有序，map 随机 | List 有序，HashMap 无序 |
| channel | 支持 | 无直接等价 |

Java 的增强 for 只给值，索引需要自己维护；Go 的 `range` 直接提供索引。Java 的 `String` 按 UTF-16 单元索引，Go 的字符串按 UTF-8 字节索引，处理 Unicode 时要特别小心。

## 动手练习

1. 用 `range` 计算切片元素之和，并打印等于目标值的索引。
2. 用 `range` 遍历 map，分别打印键值对和只打印键。
3. 遍历字符串 `"Hello, 世界"`，打印字节索引和字符。
4. 用 `range` 遍历整数 `5`，打印 `0` 到 `4`。
5. 修改切片元素的练习：先用值副本失败一次，再用索引成功修改。
6. 创建 nil 切片和 nil map，验证 `range` 不会 panic。

## 小结

- 数组/切片 `range` 返回索引和值副本；修改值副本不影响原数据。
- map `range` 返回键和值，顺序随机；只写一个变量得到键。
- 字符串 `range` 返回字节索引和 rune，不是字符下标。
- channel 可以用 `range` 读取到关闭；整数 `range` 需要 Go 1.22+。
- nil 切片和 nil map 可以安全遍历。

---

上一篇：[[Go/Advanced/Recursion|Go 进阶：递归]]
下一篇：[[Go/Advanced/Pointers|Go 进阶：指针]]

返回 [[Go/Advanced/index|Go 进阶]]。
