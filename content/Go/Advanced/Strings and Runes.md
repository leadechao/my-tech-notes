---
title: "Go 进阶：字符串与 rune"
description: "从 Go by Example 的 Strings and Runes 示例出发，理解字符串的 UTF-8 字节本质、rune、len 与字符数、range 解码和常见编码陷阱。"
tags:
  - Go
  - 编程语言
  - 进阶
  - 字符串
created: 2026-10-29
modified: 2026-10-29
---

# Go 进阶：字符串与 rune

Go 的字符串不是“字符数组”，而是**只读的字节序列**。文本通常按 UTF-8 编码存储，一个 Unicode 字符可能占多个字节。

Go 用 `rune` 表示一个 Unicode 码点。理解“字节、rune、字符串”三者的关系，是正确处理中文、emoji 和其他多字节文本的前提。

[Go by Example 的 Strings and Runes](https://gobyexample.com/strings-and-runes) 用泰文演示了字节与字符的差异。本文用中文例子说明同样的概念。

## 完整示例

```go
package main

import (
	"fmt"
	"unicode/utf8"
)

func main() {
	const s = "你好, 世界"

	fmt.Println("Len:", len(s))

	for i := 0; i < len(s); i++ {
		fmt.Printf("%x ", s[i])
	}
	fmt.Println()

	fmt.Println("Rune count:", utf8.RuneCountInString(s))

	for idx, r := range s {
		fmt.Printf("%#U starts at %d\n", r, idx)
	}

	fmt.Println("\nUsing DecodeRuneInString")
	for i, w := 0, 0; i < len(s); i += w {
		r, width := utf8.DecodeRuneInString(s[i:])
		fmt.Printf("%#U starts at %d\n", r, i)
		w = width
		examineRune(r)
	}
}

func examineRune(r rune) {
	if r == '你' {
		fmt.Println("found ni")
	} else if r == '好' {
		fmt.Println("found hao")
	}
}
```

运行：

```bash
$ go run strings-and-runes.go
Len: 14
e4 bd a0 e5 a5 bd 2c 20 e4 b8 96 e7 95 8c
Rune count: 6
U+4F60 '你' starts at 0
U+597D '好' starts at 3
U+002C ',' starts at 6
U+0020 ' ' starts at 7
U+4E16 '世' starts at 8
U+754C '界' starts at 11
...
```

## 字符串是只读字节序列

```go
const s = "你好, 世界"
fmt.Println(len(s)) // 14
```

`len(s)` 返回字节数，不是字符数。`"你好, 世界"` 有 6 个 Unicode 码点，但 UTF-8 编码占 14 个字节。

可以通过下标读取单个字节：

```go
fmt.Println(s[0]) // 228
fmt.Printf("%x\n", s[0]) // e4
```

下标访问返回 `byte`（`uint8`），不是字符。字符串不可变，不能写 `s[0] = 'x'`；需要修改时，先转换成 `[]byte` 或 `[]rune`。

## rune：一个 Unicode 码点

`rune` 是 `int32` 的别名，表示一个 Unicode 码点：

```go
var r rune = '你'
fmt.Println(r)        // 20320
fmt.Printf("%c\n", r) // 你
```

单引号包裹的是 rune 字面量，双引号包裹的是字符串：

```go
'你'   // rune
"你"   // string，占 3 个字节
```

rune 的值是码点编号，不是字节值。`'你'` 的码点是 U+4F60，十进制 20320。

## 用 `range` 解码字符串

```go
for idx, r := range s {
	fmt.Printf("%#U starts at %d\n", r, idx)
}
```

输出：

```text
U+4F60 '你' starts at 0
U+597D '好' starts at 3
U+002C ',' starts at 6
U+0020 ' ' starts at 7
U+4E16 '世' starts at 8
U+754C '界' starts at 11
```

第一个返回值是 rune 的起始**字节索引**，第二个返回值是 `rune`。

索引不连续，因为每个汉字占 3 个字节。不要把字符串索引当成字符位置：

```go
s[3] // 不是第二个字符，而是“好”的第一个字节
```

## 统计字符数

`len(s)` 统计字节数。统计 Unicode 码点数量要用 `utf8.RuneCountInString`：

```go
fmt.Println(len(s))                        // 14
fmt.Println(utf8.RuneCountInString(s))     // 6
```

`RuneCountInString` 需要逐个解码 UTF-8，因此时间复杂度与字符串长度相关。

## 手动解码：`DecodeRuneInString`

`utf8.DecodeRuneInString(s)` 返回两个值：

- 第一个是解码出的 `rune`；
- 第二个是该 rune 占用的字节数（width）。

```go
for i, w := 0, 0; i < len(s); i += w {
	r, width := utf8.DecodeRuneInString(s[i:])
	fmt.Printf("%#U starts at %d\n", r, i)
	w = width
}
```

这个循环和 `range` 的效果类似，但把解码过程显式写了出来。遇到非法 UTF-8 字节时，`DecodeRuneInString` 会返回 `utf8.RuneError`（U+FFFD），宽度为 1。

## 字节、rune 与字符串转换

### 字符串转字节切片

```go
b := []byte("你好")
fmt.Println(len(b)) // 6
```

`[]byte(s)` 会复制字节。修改 `b` 不会影响原字符串。

### 字符串转 rune 切片

```go
r := []rune("你好")
fmt.Println(len(r)) // 2
fmt.Println(r[0])   // 20320
```

`[]rune(s)` 会按 UTF-8 解码并复制码点。适合按“字符”处理字符串：

```go
runes := []rune(s)
runes[0] = '您'
fmt.Println(string(runes)) // 您好, 世界
```

### rune 转字符串

```go
fmt.Println(string('你'))       // 你
fmt.Println(string(rune(20320))) // 你
```

### 整数转字符串的陷阱

```go
fmt.Println(string(65))       // A
fmt.Println(strconv.Itoa(65)) // 65
```

`string(65)` 把整数当作 Unicode 码点，得到字符 `A`；要得到十进制文本，使用 `strconv.Itoa`。

## 子串与字节边界

`s[i:j]` 按字节截取。如果边界落在多字节字符中间，会得到非法 UTF-8：

```go
s := "你好"
fmt.Println(s[0:1]) // 输出一个非法字节，不是“你”
```

按字符截取应先转成 `[]rune`：

```go
r := []rune("你好, 世界")
fmt.Println(string(r[0:2])) // 你好
```

这是处理中文、日文、emoji 时最常见的错误之一。

## 字符串不可变

字符串一旦创建就不能修改：

```go
s := "hello"
s[0] = 'H'
// cannot assign to s[0] (neither addressable nor a map index expression)
```

拼接会创建新字符串：

```go
s = s + " world"
```

大量拼接时，使用 `strings.Builder` 可以减少内存分配：

```go
var b strings.Builder
for i := 0; i < 1000; i++ {
	b.WriteString("x")
}
s := b.String()
```

## 常见错误

### 1. 用 `len` 统计字符数

`len` 是字节数。字符数用 `utf8.RuneCountInString` 或 `len([]rune(s))`。

### 2. 用索引访问字符

`s[i]` 返回字节。遍历字符用 `range`，或转成 `[]rune`。

### 3. 按字节截取多字节字符串

`s[0:1]` 可能截断 UTF-8 字符。按字符处理时使用 `[]rune`。

### 4. `string(65)` 得到 `"A"`

这是码点转换，不是数字格式化。使用 `strconv.Itoa`。

### 5. 试图修改字符串下标

字符串不可变。转换成 `[]byte` 或 `[]rune`，修改后再转回 `string`。

### 6. 混淆 `rune` 和 `byte`

`byte` 是 0 到 255 的字节，`rune` 是 Unicode 码点。处理 ASCII 时两者看起来相似，处理中文时差异明显。

## 和 Java 对照

| 概念 | Go | Java |
| --- | --- | --- |
| 字符串编码 | UTF-8 字节序列 | UTF-16 代码单元序列 |
| 长度 | `len` 返回字节数 | `length()` 返回 UTF-16 单元数 |
| 字符类型 | `rune`（int32 码点） | `char`（16 位 UTF-16 单元） |
| 遍历字符 | `range` 自动解码 rune | `codePoints()` / `codePointAt` |
| 索引 | 字节索引 | UTF-16 单元索引 |
| 不可变 | 是 | 是 |
| 数字转字符串 | `strconv.Itoa` | `String.valueOf` |

Java 的 `String.length()` 对 BMP 之外的字符也可能返回 2，因为一个码点需要两个 UTF-16 单元；Go 的 `len` 对多字节 UTF-8 字符返回更多字节。两者都不能简单地把“长度”当字符数。

## 动手练习

1. 打印 `"你好, 世界"` 的字节长度和 rune 数量。
2. 用 `range` 遍历字符串，打印每个字符的字节索引和码点。
3. 用 `utf8.DecodeRuneInString` 手动解码字符串。
4. 把字符串转成 `[]rune`，反转后转回字符串。
5. 比较 `string(65)` 和 `strconv.Itoa(65)` 的输出。
6. 按字节截取一个中文字符串，观察是否产生非法 UTF-8，再用 `[]rune` 修复。

## 小结

- 字符串是只读的 UTF-8 字节序列，`len` 返回字节数。
- `rune` 表示 Unicode 码点，单引号是 rune 字面量。
- `range` 遍历字符串时返回字节索引和 rune。
- 统计字符数用 `utf8.RuneCountInString`，按字符截取用 `[]rune`。
- `string(int)` 是码点转换，数字转文本用 `strconv.Itoa`。
- 字符串不可变，修改需要先转换。

---

上一篇：[[Go/Advanced/Pointers|Go 进阶：指针]]
下一篇：[[Go/Advanced/Structs|Go 进阶：结构体]]

返回 [[Go/Advanced/index|Go 进阶]]。
