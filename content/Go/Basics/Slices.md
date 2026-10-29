---
title: "Go 基础：切片"
description: "从 Go by Example 的 Slices 示例出发，理解 nil 切片、make、append、copy、切片表达式、底层数组共享与 slices 工具包。"
tags:
  - Go
  - 编程语言
  - 基础
  - 切片
created: 2026-10-29
modified: 2026-10-29
---

# Go 基础：切片

切片（slice）是 Go 中最常用的序列类型。它比数组灵活：长度可以变化，可以追加、截取和复制。但灵活也带来了几个必须理解的概念：长度、容量、底层数组和共享。

[Go by Example 的 Slices](https://gobyexample.com/slices) 覆盖了创建、追加、复制、截取和二维切片。本文补充 nil 与空切片的区别、append 的别名问题，以及 `slices` 工具包的常用函数。

## 完整示例

```go
package main

import (
	"fmt"
	"slices"
)

func main() {
	var s []string
	fmt.Println("uninit:", s, s == nil, len(s) == 0)

	s = make([]string, 3)
	fmt.Println("emp:", s, "len:", len(s), "cap:", cap(s))

	s[0] = "a"
	s[1] = "b"
	s[2] = "c"
	fmt.Println("set:", s)
	fmt.Println("get:", s[2])

	fmt.Println("len:", len(s))

	s = append(s, "d")
	s = append(s, "e", "f")
	fmt.Println("apd:", s)

	c := make([]string, len(s))
	copy(c, s)
	fmt.Println("cpy:", c)

	l := s[2:5]
	fmt.Println("sl1:", l)

	l = s[:5]
	fmt.Println("sl2:", l)

	l = s[2:]
	fmt.Println("sl3:", l)

	t := []string{"g", "h", "i"}
	fmt.Println("dcl:", t)

	t2 := []string{"g", "h", "i"}
	if slices.Equal(t, t2) {
		fmt.Println("t == t2")
	}

	twoD := make([][]int, 3)
	for i := range 3 {
		innerLen := i + 1
		twoD[i] = make([]int, innerLen)
		for j := range innerLen {
			twoD[i][j] = i + j
		}
	}
	fmt.Println("2d: ", twoD)
}
```

运行：

```bash
$ go run slices.go
uninit: [] true true
emp: [  ] len: 3 cap: 3
set: [a b c]
get: c
len: 3
apd: [a b c d e f]
cpy: [a b c d e f]
sl1: [c d e]
sl2: [a b c d e]
sl3: [c d e f]
dcl: [g h i]
t == t2
2d:  [[0] [1 2] [2 3 4]]
```

## nil 切片与空切片

切片类型写成 `[]T`，长度不写在类型里。未初始化的切片是 `nil`：

```go
var s []string
fmt.Println(s == nil, len(s) == 0) // true true
```

nil 切片的长度和容量都是 0，可以直接 `append`：

```go
var s []int
s = append(s, 1)
fmt.Println(s) // [1]
```

空切片和 nil 切片不同：

```go
var a []int        // nil
b := []int{}       // 非 nil，长度 0
c := make([]int, 0) // 非 nil，长度 0

fmt.Println(a == nil) // true
fmt.Println(b == nil) // false
fmt.Println(c == nil) // false
```

三者长度都是 0，但 nil 切片没有分配底层数组。JSON 编码、反射和某些库的语义可能会区分 nil 和空切片，需要根据场景选择。

## 用 `make` 创建切片

`make` 可以指定长度，也可以同时指定容量：

```go
s := make([]string, 3)        // len=3, cap=3
s2 := make([]int, 2, 5)       // len=2, cap=5
```

长度是当前可访问的元素个数，容量是底层数组从切片起始位置开始可用的元素个数。容量必须大于等于长度：

```go
fmt.Println(len(s), cap(s)) // 3 3
```

用 `make` 创建的切片，元素会被初始化为零值；`make([]string, 3)` 得到三个空字符串。

## 读写元素与长度

和数组一样，切片通过下标读写：

```go
s := make([]string, 3)
s[0] = "a"
s[1] = "b"
s[2] = "c"

fmt.Println(s[2]) // c
fmt.Println(len(s)) // 3
```

下标必须在 `[0, len(s))` 范围内，越界会 panic。切片不能通过下标自动扩容。

## `append`：追加元素

`append` 返回一个新的切片，必须把返回值赋回去：

```go
s = append(s, "d")
s = append(s, "e", "f")
```

也可以追加另一个切片：

```go
s = append(s, other...)
```

如果底层数组还有足够容量，`append` 会直接在原数组上写入；容量不够时，会分配更大的数组，把旧元素复制过去。因此 `append` 可能复用底层数组，也可能产生新数组，不能假设它一定复制或一定不复制。

> [!warning]
> 不要忽略 `append` 的返回值。`append(s, x)` 本身不会改变 `s` 的长度；只有把返回的切片赋回 `s`，新元素才“属于”这个切片。

## `copy`：复制元素

`copy(dst, src)` 把 `src` 的元素复制到 `dst`，返回实际复制的元素个数：

```go
c := make([]string, len(s))
n := copy(c, s)
fmt.Println(n, c)
```

复制的数量是 `min(len(dst), len(src))`。如果目标切片太短，只会复制前面一部分：

```go
dst := make([]int, 2)
n := copy(dst, []int{1, 2, 3})
fmt.Println(n, dst) // 2 [1 2]
```

`copy` 复制的是元素值。如果元素是指针或包含指针的结构体，复制后仍然共享指向的对象。

## 切片表达式

切片支持 `s[low:high]`，结果包含 `low`，不包含 `high`：

```go
s := []string{"a", "b", "c", "d", "e", "f"}

fmt.Println(s[2:5]) // [c d e]
fmt.Println(s[:5])  // [a b c d e]
fmt.Println(s[2:])  // [c d e f]
```

省略 `low` 时默认为 0，省略 `high` 时默认为 `len(s)`。

切片表达式的结果和原切片共享底层数组：

```go
a := []int{1, 2, 3, 4}
b := a[1:3]
b[0] = 99

fmt.Println(a) // [1 99 3 4]
fmt.Println(b) // [99 3]
```

修改 `b` 的元素会影响 `a`，因为它们指向同一块内存。切片不是深拷贝。

> [!note]
> 对切片做切片时，`high` 的上限是 `cap(s)`，不是 `len(s)`。例如 `s := make([]int, 2, 5)` 时，`s[:4]` 是合法的，结果是长度 4、容量 5 的切片。这允许重新利用底层数组的剩余容量，但也容易读到未预期的旧数据或零值。日常代码中建议把 `high` 限制在 `len(s)` 内。

## 完整切片表达式

三索引形式 `s[low:high:max]` 可以同时限制结果的容量：

```go
s := []int{0, 1, 2, 3, 4}
t := s[1:3:3] // len=2, cap=2

u := append(t, 99)
fmt.Println(s) // [0 1 2 3 4]
fmt.Println(t) // [1 2]
fmt.Println(u) // [1 2 99]
```

因为 `t` 的容量被限制为 2，`append` 必须分配新数组，所以不会覆盖 `s` 的后续元素。这在需要隔离底层数组时很有用。

## `append` 的别名陷阱

```go
a := []int{1, 2, 3}
b := a[:2]
b = append(b, 99)

fmt.Println(a) // [1 2 99]
fmt.Println(b) // [1 2 99]
```

`a` 的长度是 3，容量也是 3；`b := a[:2]` 的长度是 2，容量是 3。`append(b, 99)` 还有容量可用，于是直接写入 `a[2]`，导致 `a` 也被修改。

如果希望 `b` 完全独立，可以先复制：

```go
b := make([]int, 2)
copy(b, a[:2])
b = append(b, 99)
```

或者使用 `slices.Clone`。

## `slices` 工具包

Go 1.21 引入了 `slices` 包，提供常用的切片操作：

```go
import "slices"

fmt.Println(slices.Equal(t, t2)) // 比较内容
c := slices.Clone(t)             // 浅拷贝
fmt.Println(slices.Contains(t, "g"))
slices.Sort(t)
```

`Equal` 比较长度和每个元素；`Clone` 返回一个独立的浅拷贝；`Contains` 判断是否包含某个元素；`Sort` 对切片排序。完整 API 可以查阅标准库文档。

`slices.Clone` 是浅拷贝：如果元素是指针，克隆后的切片仍然指向同一批对象。

## 二维切片

二维切片是“切片的切片”，内层长度可以不同：

```go
twoD := make([][]int, 3)
for i := range 3 {
	innerLen := i + 1
	twoD[i] = make([]int, innerLen)
	for j := range innerLen {
		twoD[i][j] = i + j
	}
}
fmt.Println(twoD) // [[0] [1 2] [2 3 4]]
```

`make([][]int, 3)` 只创建了 3 个 nil 行，每行还需要单独 `make`。这和二维数组 `[2][3]int` 的矩形结构不同。

## 用 `range` 遍历切片

```go
for i, v := range s {
	fmt.Println(i, v)
}
```

`v` 是元素的副本。修改 `v` 不会修改切片；要修改元素，使用下标：

```go
for i := range s {
	s[i] = strings.ToUpper(s[i])
}
```

如果只需要值，可以用 `_` 忽略索引；如果只需要索引，省略第二个变量即可。

## 常见错误

### 1. 忽略 `append` 的返回值

```go
append(s, "x") // 错误：结果被丢弃
```

必须写成 `s = append(s, "x")`。

### 2. 以为切片表达式会复制

`s[low:high]` 共享底层数组。修改子切片会影响原切片。

### 3. append 覆盖了共享数组

如上面的别名陷阱所示，容量足够时 `append` 可能修改原切片。需要隔离时先复制。

### 4. 下标越界

访问 `s[len(s)]` 会 panic。追加元素应使用 `append`，不要手动越界赋值。

### 5. `copy` 的目标太短

`copy` 只复制 `min(len(dst), len(src))` 个元素。目标容量不足时不会自动扩容。

### 6. 混淆 nil 切片与空切片

两者长度都是 0，但 `nil` 切片等于 `nil`，空切片不等于 `nil`。在 JSON、接口和数据库场景中要留意。

### 7. 直接用 `==` 比较切片

切片只能和 `nil` 比较，不能比较两个切片的内容。使用 `slices.Equal`。

## 和 Java 对照

| 概念 | Go 切片 | Java |
| --- | --- | --- |
| 类型 | `[]T`，长度运行时可变 | 数组 `T[]` 固定；`ArrayList<T>` 可变 |
| 赋值 | 复制切片头，共享底层数组 | 数组复制引用；`ArrayList` 复制引用 |
| 截取 | `s[low:high]`，共享底层数组 | `Arrays.copyOfRange` 复制；`subList` 是视图 |
| 追加 | `append`，返回新切片 | `ArrayList.add` |
| 长度/容量 | `len` / `cap` | `size` / 内部容量 |
| 内容比较 | `slices.Equal` | `Arrays.equals` / `List.equals` |
| nil | nil 切片 | null 引用 |

Go 的切片更像“数组指针 + 长度 + 容量”的描述符。赋值时复制的是描述符，底层数组仍然共享，这是理解切片行为的关键。

## 动手练习

1. 声明 nil 切片和空切片，比较它们与 `nil` 的关系。
2. 用 `make([]int, 3, 5)` 创建切片，打印长度和容量，再 append 两个元素，观察容量变化。
3. 用 `copy` 复制一个切片，修改副本，验证原切片不变。
4. 用 `s[1:3]` 截取切片，修改子切片，观察原切片变化。
5. 用三索引切片限制容量，验证 append 不会覆盖原数组。
6. 创建二维切片，让每一行长度分别为 1、2、3。

## 小结

- 切片由底层数组、长度和容量组成，长度可变。
- nil 切片长度和容量为 0，可以直接 append；空切片非 nil。
- `append` 可能复用底层数组，也可能分配新数组，必须接收返回值。
- 切片表达式共享底层数组，修改子切片会影响原切片。
- 三索引切片可以限制容量，避免 append 意外覆盖。
- `slices` 包提供 Equal、Clone、Contains、Sort 等工具。

---

上一篇：[[Go/Basics/Arrays|Go 基础：数组]]
下一篇：[[Go/Basics/Maps|Go 基础：map]]

返回 [[Go/Basics/index|Go 基础]]。
