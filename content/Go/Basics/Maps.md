---
title: "Go 基础：map"
description: "从 Go by Example 的 Maps 示例出发，掌握 map 的创建、读写、comma-ok、delete、clear、遍历顺序、nil map 与 maps 工具包。"
tags:
  - Go
  - 编程语言
  - 基础
  - map
created: 2026-10-29
modified: 2026-10-29
---

# Go 基础：map

map 是 Go 内置的关联容器，用来保存键值对。其他语言里可能叫字典（dict）、哈希表（hash）或关联数组。

map 的读写非常直接，但有几个和切片不同的地方：键必须可比较、缺失键返回零值、遍历顺序随机、nil map 可读不可写。理解这些边界，才能避免运行时 panic。

[Go by Example 的 Maps](https://gobyexample.com/maps) 覆盖了创建、读写、删除、清空和比较。本文补充 comma-ok、引用语义、键类型限制和并发注意事项。

## 完整示例

```go
package main

import (
	"fmt"
	"maps"
)

func main() {
	m := make(map[string]int)

	m["k1"] = 7
	m["k2"] = 13
	fmt.Println("map:", m)

	v1 := m["k1"]
	fmt.Println("v1:", v1)

	v3 := m["k3"]
	fmt.Println("v3:", v3)

	fmt.Println("len:", len(m))

	delete(m, "k2")
	fmt.Println("map:", m)

	clear(m)
	fmt.Println("map:", m)

	_, prs := m["k2"]
	fmt.Println("prs:", prs)

	n := map[string]int{"foo": 1, "bar": 2}
	fmt.Println("map:", n)

	n2 := map[string]int{"foo": 1, "bar": 2}
	if maps.Equal(n, n2) {
		fmt.Println("n == n2")
	}
}
```

运行（`fmt` 打印 map 时通常会按 key 排序，便于阅读）：

```bash
$ go run maps.go
map: map[k1:7 k2:13]
v1: 7
v3: 0
len: 2
map: map[k1:7]
map: map[]
prs: false
map: map[bar:2 foo:1]
n == n2
```

## 创建 map

用 `make` 创建空 map：

```go
m := make(map[string]int)
```

类型 `map[string]int` 表示“键是 string，值是 int”。也可以直接使用字面量：

```go
n := map[string]int{"foo": 1, "bar": 2}
```

零值 map 是 `nil`：

```go
var m map[string]int
fmt.Println(m == nil) // true
```

nil map 可以读取、`len`、`delete` 和 `range`，但不能写入。写入前必须用 `make` 或字面量初始化：

```go
var m map[string]int
m["x"] = 1
// panic: assignment to entry in nil map
```

> [!warning]
> 这是 map 和切片的一个重要差异：nil 切片可以直接 `append`，nil map 不能直接写入。声明 map 后如果准备写入，记得先 `make`。

## 读写键值

设置和读取都使用下标语法：

```go
m := make(map[string]int)

m["k1"] = 7
m["k2"] = 13

fmt.Println(m["k1"]) // 7
```

读取不存在的键不会报错，而是返回值类型的零值：

```go
fmt.Println(m["missing"]) // 0
```

如果值类型是 `string`，零值是 `""`；如果是指针，零值是 `nil`。这意味着仅凭返回值无法判断键是否存在。

## comma-ok：判断键是否存在

使用两个返回值的形式，第二个返回值表示键是否存在：

```go
v, ok := m["k1"]
fmt.Println(v, ok) // 7 true

v, ok = m["missing"]
fmt.Println(v, ok) // 0 false
```

如果只关心键是否存在，可以用 `_` 忽略值：

```go
_, exists := m["k2"]
if !exists {
	fmt.Println("k2 not found")
}
```

comma-ok 能区分“键不存在”和“键存在但值刚好是零值”：

```go
m["zero"] = 0

v1, ok1 := m["zero"]
v2, ok2 := m["missing"]

fmt.Println(v1, ok1) // 0 true
fmt.Println(v2, ok2) // 0 false
```

## 长度、删除与清空

`len` 返回键值对数量：

```go
fmt.Println(len(m)) // 2
```

`delete` 删除指定键：

```go
delete(m, "k2")
```

删除不存在的键不会报错，也不会 panic。

`clear` 清空整个 map：

```go
clear(m)
fmt.Println(len(m)) // 0
```

`clear` 是 Go 1.21 引入的内置函数。对 nil map 调用 `clear` 也是安全的。

## 比较 map

map 不能使用 `==` 比较内容，只能和 `nil` 比较：

```go
a := map[string]int{"x": 1}
b := map[string]int{"x": 1}

// a == b
// invalid operation: a == b (map can only be compared to nil)
```

需要比较内容时，使用 `maps.Equal`：

```go
if maps.Equal(a, b) {
	fmt.Println("same")
}
```

`maps.Equal` 要求键和值都可比较。对于值类型是切片、map 等不可比较类型的情况，需要自己实现比较逻辑。

## 遍历顺序是随机的

```go
for k, v := range m {
	fmt.Println(k, v)
}
```

map 的遍历顺序没有保证，每次运行可能不同。不要依赖顺序编写业务逻辑；需要有序输出时，先取出键并排序：

```go
keys := make([]string, 0, len(m))
for k := range m {
	keys = append(keys, k)
}
slices.Sort(keys)

for _, k := range keys {
	fmt.Println(k, m[k])
}
```

`fmt.Println` 打印 map 时通常会按键排序，但这是格式化输出的行为，不代表遍历有序。

## 引用语义与复制

map 是引用类型。把一个 map 赋给另一个变量，两个变量共享同一份底层数据：

```go
m := map[string]int{"a": 1}
n := m

n["b"] = 2
fmt.Println(m) // map[a:1 b:2]
```

如果需要一个独立副本，可以使用 `maps.Clone`：

```go
clone := maps.Clone(m)
clone["c"] = 3

fmt.Println(m)     // 原来的 map 不受影响
fmt.Println(clone)
```

`maps.Clone` 是浅拷贝：如果值是指针或包含指针的结构体，克隆后仍然共享指向的对象。

## 键必须可比较

map 的键类型必须可比较。布尔、数值、字符串、指针、接口，以及由可比较字段组成的结构体和数组都可以作为键。切片、map、函数不能作为键：

```go
m := map[[]int]string{}
// invalid map key type []int
```

接口类型虽然可以作为键，但运行时的动态值也必须可比较；如果放入切片这类不可比较的值，会触发 panic。

如果需要用切片作为键，可以先把切片转换成字符串（例如编码后），或者使用自定义的哈希函数。

## 修改 map 中的结构体值

map 的下标表达式不可取地址，因此不能直接修改结构体值的字段：

```go
type Point struct{ X, Y int }

m := map[string]Point{"a": {1, 2}}
m["a"].X = 3
// cannot assign to struct field m["a"].X in map
```

正确做法是取出、修改、再放回：

```go
p := m["a"]
p.X = 3
m["a"] = p
```

如果值是指针，则可以直接通过指针修改字段。

## 并发访问

Go 的内置 map 不是并发安全的。多个 goroutine 同时读写同一个 map 会触发数据竞争，甚至直接 panic：

```text
fatal error: concurrent map read and map write
```

并发场景下应使用：

- `sync.RWMutex` 保护 map；
- `sync.Map`（适合读多写少或键集合稳定的场景）；
- 分片锁或其他并发数据结构。

仅仅使用 `sync.Map` 并不总是更快，普通 map 加锁通常已经足够。

## 常见错误

### 1. 向 nil map 写入

声明后没有 `make` 就写入，会 panic。读取、`len`、`delete`、`range` 是安全的。

### 2. 使用不可比较的类型作为键

切片、map、函数不能作为键。需要先转换成可比较的类型。

### 3. 依赖遍历顺序

map 的 `range` 顺序随机。需要顺序时先排序键。

### 4. 直接修改结构体字段

map 下标不可取地址，结构体值需要复制、修改、再赋值。

### 5. 用 `==` 比较两个 map

map 只能和 `nil` 比较。内容比较使用 `maps.Equal`。

### 6. 把缺失键的零值当成“存在”

需要区分时使用 comma-ok。

### 7. 并发读写 map

内置 map 不支持并发写。使用锁或并发安全的数据结构。

## 和 Java 对照

| 概念 | Go | Java |
| --- | --- | --- |
| 类型 | `map[K]V` | `Map<K,V>` 接口，常用 `HashMap` |
| 创建 | `make` 或字面量 | `new HashMap<>()` |
| 缺失键 | 返回零值 | 返回 `null` |
| 判断存在 | `v, ok := m[k]` | `containsKey` / `get` |
| 删除 | `delete(m, k)` | `remove(k)` |
| 清空 | `clear(m)` | `clear()` |
| 内容比较 | `maps.Equal` | `equals` |
| 遍历顺序 | 随机 | `HashMap` 不保证，`LinkedHashMap` 保持插入顺序 |
| 并发安全 | 需额外加锁 | `ConcurrentHashMap` |

Java 的 `HashMap.get` 返回 `null` 表示键不存在，但值本身也可能是 `null`；Go 的 map 返回值类型的零值，同样需要用 comma-ok 区分。两者都需要注意“缺失”和“零值”的歧义。

## 动手练习

1. 用 map 统计一段文本中每个单词出现的次数。
2. 用 comma-ok 判断某个键是否存在，并区分值为零值的情况。
3. 创建 map，依次执行 delete 和 clear，观察长度变化。
4. 用 `maps.Equal` 比较两个内容相同的 map。
5. 用 `maps.Clone` 创建副本，修改副本，验证原 map 不变。
6. 用 map 保存结构体值，尝试直接修改字段，再用“取出-修改-放回”修复。
7. 用 `sync.RWMutex` 保护一个 map，在多个 goroutine 中并发读写。

## 小结

- map 是键值对容器，键类型必须可比较。
- 用 `make` 或字面量创建；nil map 可读不可写。
- 缺失键返回零值，用 comma-ok 判断键是否存在。
- `len`、`delete`、`clear` 是常用操作；内容比较用 `maps.Equal`。
- 遍历顺序随机，需要顺序时先排序键。
- map 是引用类型，赋值共享底层数据；需要独立副本时使用 `maps.Clone`。
- 内置 map 不是并发安全的，并发场景需要加锁或使用并发安全结构。

---

上一篇：[[Go/Basics/Slices|Go 基础：切片]]
下一篇：[Go by Example: Functions](https://gobyexample.com/functions)

返回 [[Go/Basics/index|Go 基础]]。
