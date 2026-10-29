---
title: "Go 入门：从 Hello World 开始"
description: "以 Go by Example 的 Hello World 为例，讲清 Go 程序的最小结构、go run 与 go build 的区别，以及初学者最容易踩的坑。"
tags:
  - Go
  - 编程语言
  - 基础
  - Hello World
created: 2026-10-29
modified: 2026-10-29
---

# Go 入门：从 Hello World 开始

学习一门语言，最合适的起点永远是那个能打印一行字的程序。它足够小，小到可以逐行拆开；又足够完整，完整到能跑通“编写 → 编译 → 运行”的全过程。

本文以 [Go by Example 的 Hello World](https://gobyexample.com/hello-world) 为蓝本，补上原页没有展开的细节，并穿插一些从 Java 转 Go 时容易困惑的地方。

## 第一个 Go 程序

新建文件 `hello-world.go`，写入：

```go
package main

import "fmt"

func main() {
	fmt.Println("hello world")
}
```

运行：

```bash
$ go run hello-world.go
hello world
```

只有四行有效代码，却包含了 Go 程序最基本的四个组成部分：包声明、导入、入口函数和语句。

> [!note]
> Go 不强制使用制表符或空格缩进，但官方工具 `gofmt` 统一使用制表符。实际开发中不需要手动纠结格式，保存前执行 `gofmt -w hello-world.go`，或直接依赖编辑器的“保存时格式化”即可。

## 逐行拆解

### `package main`：声明可执行包

每个 `.go` 文件都必须以包声明开始（注释和空行可以出现在它前面）。Go 用包来组织代码，一个目录通常对应一个包。

包名有两种常见的角色：

- `main`：特殊包名，表示这是一个可执行程序，编译后能得到二进制文件。
- 其他包名（如 `utils`、`server`）：普通库包，供其他代码导入，本身不能直接运行。

可执行程序必须同时满足两个条件：**包名是 `main`**，并且**定义了 `func main()`**。二者缺一不可。

同一个目录下的普通 `.go` 文件必须属于同一个包；测试文件 `_test.go` 可以使用 `包名_test` 的形式，这是例外。

### `import "fmt"`：导入标准库

`fmt` 是 Go 标准库中的格式化输入输出包，`Println`、`Printf`、`Sprintf` 都在这里。导入路径是字符串字面量：

```go
import "fmt"
```

需要导入多个包时，使用括号分组：

```go
import (
	"fmt"
	"os"
)
```

Go 对导入有一个很“硬”的规则：**导入了却没用，编译直接失败**。

```text
./hello-world.go:3:8: "fmt" imported and not used
```

这不是警告，而是错误。少一个 import 会报 `undefined`，多一个 import 会报“imported and not used”，编译器逼着代码保持干净。

### `func main()`：程序入口

```go
func main() {
	fmt.Println("hello world")
}
```

`main` 函数是程序启动后执行的第一段代码。它的签名是固定的：

- 不能有参数；
- 不能有返回值。

写成 `func main(args []string)` 会得到：

```text
func main must have no arguments and no return values
```

命令行参数需要通过 `os.Args` 获取；需要返回退出码时，使用 `os.Exit`。

函数名首字母的大小写还决定了可见性：大写开头是导出的（包外可见），小写开头是包内私有的。`main` 虽然是小写，但它是运行时约定的特殊入口，不受这条规则影响。

### `fmt.Println("hello world")`：输出一行

`Println` 的 `ln` 表示 line，它会做两件事：

1. 在多个参数之间自动插入空格；
2. 在末尾追加换行符。

```go
fmt.Println("hello", "world") // hello world
fmt.Print("hello")            // hello（不换行）
fmt.Printf("%s\n", "hello")   // hello（格式化输出）
```

`Println` 的完整签名是 `func Println(a ...any) (n int, err error)`，返回写入的字节数和可能的错误。打印到终端时通常不会失败，所以直接忽略返回值即可。

字符串使用双引号。如果需要原样保留换行和反斜杠，可以使用反引号包裹的原始字符串：

```go
msg := `第一行
第二行`
fmt.Println(msg)
```

## `go run` 与 `go build`

Go 提供了两条最常用的命令，它们解决的是不同场景的问题。

### 用 `go run` 快速验证

```bash
$ go run hello-world.go
hello world
```

`go run` 会在后台完成编译、链接，然后把生成的临时可执行文件跑起来。它**不会在当前目录留下二进制文件**，适合写代码时反复验证。

虽然叫“直接运行”，但它并不是解释执行：代码仍然要经过完整的编译，语法错误、类型错误一个都不会漏掉。

### 用 `go build` 生成可执行文件

```bash
$ go build hello-world.go
$ ls
hello-world    hello-world.go
$ ./hello-world
hello world
```

`go build hello-world.go` 会在当前目录生成名为 `hello-world` 的可执行文件（Windows 下是 `hello-world.exe`），之后可以反复运行，也可以复制到同架构的机器上部署。

用 `-o` 可以指定输出文件名：

```bash
$ go build -o hello hello-world.go
$ ./hello
hello world
```

一个容易忽略的细节：如果当前包不是 `main` 包，`go build` 只做编译检查，不会产生可执行文件。

### 两者对比

| 对比项 | `go run` | `go build` |
| --- | --- | --- |
| 产物 | 临时文件，不保留 | 当前目录下的可执行文件 |
| 主要用途 | 开发阶段快速验证 | 构建、分发、部署 |
| 执行次数 | 每次执行都重新编译（有构建缓存加速） | 编译一次，可多次运行 |
| 编译错误 | 同样会报错 | 同样会报错 |

简单记：**写完想立刻看结果用 `go run`，要交付或反复运行用 `go build`。**

### 关于跨平台

Go 编译出来的是目标平台的原生二进制，不依赖 JVM，也不要求目标机器预装 Go 运行时。这让部署非常轻量，但二进制本身与操作系统、CPU 架构绑定。

需要为 Linux 服务器构建时，可以在 macOS 上交叉编译：

```bash
$ GOOS=linux GOARCH=amd64 go build hello-world.go
```

生成的 `hello-world` 就可以直接放到对应的 Linux 机器上运行。Windows PowerShell 的环境变量写法不同，跨平台构建更推荐交给 CI 或 Makefile 统一处理。

## 常见错误与排查

下面几个错误在刚上手时几乎一定会遇到。

### 1. `go: command not found`

说明 Go 没有安装，或者安装目录没有加入 `PATH`。先确认：

```bash
$ go version
```

如果命令不存在，请到 [go.dev/dl](https://go.dev/dl/) 安装，并按照安装说明配置环境变量。

### 2. `expected 'package', found ...`

文件开头不是合法的包声明。常见原因：

- 忘了写 `package main`；
- 文件开头有编辑器写入的 BOM 或其他不可见字符；
- 把一段普通代码直接粘进了空文件。

### 3. `function main is undeclared in the main package`

包名是 `main`，但找不到入口函数。检查是否把函数写成了 `Main`（大写），或者签名写错。只写 `func Main()` 并不会成为入口：

```text
runtime.main_main·f: function main is undeclared in the main package
```

### 4. `"fmt" imported and not used`

导入了 `fmt` 却没有使用。删掉 import，或者补上实际调用。Go 不允许“先导入，以后可能用”的写法。

### 5. `undefined: fmt.PrintLn`

Go 区分大小写，标准库函数名是 `Println`，不是 `PrintLn`。编译器通常会贴心地提示“but have Println”。

### 6. `go run .` 提示找不到模块

`go run hello-world.go` 直接指定了文件，可以脱离模块运行；但在项目目录里执行 `go run .` 时，Go 需要知道当前模块的路径。此时先初始化模块：

```bash
$ go mod init example.com/hello
```

然后再执行 `go run .`。

## 从单文件走向模块

单文件示例适合理解语法，但真实项目会很快需要依赖管理和版本控制。Go 1.11 之后的标准做法是使用模块（module）。

一个最小的项目结构：

```text
hello/
├── go.mod
└── main.go
```

初始化并运行：

```bash
$ mkdir hello && cd hello
$ go mod init example.com/hello
$ go run .
```

`go.mod` 记录模块路径和 Go 版本；引入第三方依赖后，还会多出 `go.sum` 用于校验。日常开发中常用的命令还有：

```bash
$ go mod tidy   # 补齐缺失依赖，移除无用依赖
$ gofmt -w .    # 格式化当前目录下的 Go 代码
$ go vet ./...  # 静态检查
```

从“能跑”到“像个项目”，通常就是这几步。

## 和 Java 对照

如果之前写 Java，下面这张表可以帮助快速建立映射：

| 概念 | Go | Java |
| --- | --- | --- |
| 入口 | `package main` + `func main()` | `public static void main(String[] args)` |
| 标准输出 | `fmt.Println` | `System.out.println` |
| 编译产物 | 原生可执行文件 | `.class` 字节码，由 JVM 执行 |
| 依赖管理 | `go.mod` / `go.sum` | Maven 的 `pom.xml` 或 Gradle |
| 代码格式 | `gofmt` 统一格式 | 依赖 IDE 或格式化插件 |
| 访问控制 | 标识符首字母大小写 | `public` / `private` 等关键字 |
| 包组织 | 一个目录一个包 | `package` 与目录结构可不对应 |

最直观的差异在部署：Java 应用通常需要目标机器安装对应版本的 JDK/JRE，而 Go 程序编译后是一个自带运行时的二进制文件，复制过去就能执行。

## 动手练习

1. 把输出改成 `Hello, Go!`，分别用 `go run` 和 `go build` 运行。
2. 用 `fmt.Printf` 输出 `你好，Go！`，体会 `%s` 和 `\n` 的作用。
3. 故意删掉 `import "fmt"`，观察编译错误；再故意写一个未使用的 import，对比报错信息。
4. 使用 `go build -o hello hello-world.go`，生成自定义名称的二进制并执行。
5. 把文件放进一个新目录，执行 `go mod init example.com/hello`，再用 `go run .` 运行。

## 小结

Hello World 虽然简单，但已经把 Go 的几个核心设计暴露出来了：

- 用 `package` 组织代码，用 `main` 包和 `main` 函数定义入口；
- 用 `import` 显式声明依赖，未使用的导入直接报错；
- 用 `go run` 快速验证，用 `go build` 生成可分发的原生二进制；
- 用 `gofmt` 统一风格，把“格式争论”从项目中移除。

下一步可以继续看 Go by Example 的 [Values](https://gobyexample.com/values)，或者从 [A Tour of Go](https://go.dev/tour/welcome/1) 系统过一遍基础语法。

## 参考资料

- [Go by Example: Hello World](https://gobyexample.com/hello-world)
- [A Tour of Go](https://go.dev/tour/welcome/1)
- [How to Write Go Code](https://go.dev/doc/code)
- [The Go Programming Language Specification](https://go.dev/ref/spec)
- [Effective Go](https://go.dev/doc/effective_go)

> [!info] 版权说明
> 本文基于 [Go by Example](https://github.com/mmcgrana/gobyexample) 的 Hello World 示例扩展撰写，原作者为 Mark McGranaghan 和 Eli Bendersky。原站示例代码采用 BSD 风格许可，其余内容采用 [CC BY 3.0](https://github.com/mmcgrana/gobyexample#license) 许可。

---

下一篇：[[Go/Basics/Values|Go 基础：值类型与字面量]]

返回 [[Go/Basics/index|Go 基础]]。
