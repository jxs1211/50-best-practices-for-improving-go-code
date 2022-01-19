33 掌握 Go 代码调试利器：delve

## 掌握 Go 代码调试利器：delve

## 1. 关于调试，你首先应该知道的几件事

提到程序员，多数人的第一反应就是编码。但我们知道软件开发可不仅仅只有编码，[**调试 (debug)**](http://en.wikipedia.org/wiki/Software_BUG) 也是每个程序员的必备技能，占据了程序员很大一部分精力。调试的目标是修正代码中确认存在的 **BUG**。BUG 是什么呢？**BUG 就是编码过程的伴生品**。既然将之诠释为 “伴生品”，那就意味着 **“凡是软件，内必有 BUG”**。也许有人不同意这样的观点，但无关大碍，因为如何看待 BUG 本身就可以看成是一个哲学范畴的话题，大可见仁见智。

### 1) 调试前，首先做好心理准备

调试 BUG 的过程可以用 “艰苦卓绝” 来形容，特别是一些 “又臭又硬” 的 BUG：难于重现、难于定位、甚至在投入相当大的精力后仍然无法修复。所以，调试 BUG 前就要摆正心态、保持清醒、保持耐心，甚至要做好打 “持久战” 的打算。要知道 Unix/Linux 下有一些 BUG 是在隐藏了几十年后才被修复的。

### 2) 预防 BUG 的发生，降低 BUG 的发生概率

BUG 可简单分为产品发布前 BUG 和发布后的 BUG。无论哪一种，你都要经历收集数据、重现 BUG、定位问题、修正问题和修正后的版本验证等多个步骤。这其中又以发布后的 BUG 花费的成本为更高。既然事实证明：**与编码相比，调试 BUG 的成本更高**，那我们何不采用一些手段来预防 BUG 的发生，降低 BUG 发生的概率呢！

从一个软件的整个生命周期来看，保证软件质量应从需求开始，但这里我们主要关注编码阶段。从个体开发者角度我们可以从以下几个方面考虑：

- 充分的代码检查

充分利用你手头上工具对你编写的代码做严格的检查，这些工具包括编译器 (尽可能将警告级别提升到你可以接受的最高级别)、静态或动态代码检查工具 (linter，如 go vet) 等。这将帮助你将代码中潜在的细小问题一一发掘出来，避免这些问题在事后成为隐藏的 BUG。

- 调试版添加断言

充分利用断言 (Assertion) 这把发现 BUG 的利器，借鉴契约式编程的一些规则，在你的调试版代码中适当的地方添加断言 (Go 没有提供原生断言机制，可以自行简单实现或利用第三方库)，这样的方法同样可以帮助你及时发现代码中隐藏的缺陷。

- 充分的单元测试

充分的单元测试提高代码的测试覆盖率，减少业务逻辑理解失误或遗失导致的 BUG；单元测试用例还可以结合断言发现更多程序潜在问题。如果能做到测试先行，效果可能会更好。

- 代码同级评审

充分授权其他人来审核你的代码，提前帮你发现潜在的问题。

从组织的角度来看，持续集成 / 交付的实践可以更加及时的发现编码阶段的问题，不让问题遗漏到后面阶段成为严重 BUG。

如果很好的实施了上述这些手段后，你的 BUG 发生率会大大降低。事实证明：近些年来，编程语言的编译器日渐强大、各种静态和动态语法检查器 (linter，如 go vet) 等工具的运用以及测试驱动开发的广泛接纳 (至少能编写更多的单元测试) 的确使得调试阶段在程序员总付出时间中的占比有所减少。

### 3) BUG 的原因定位和修正

前面说过：**BUG 不能避免**。一旦 BUG 发生了，我们该怎么办？其实与 BUG 做艰苦卓绝的斗争也是有一定方法的。

- 收集 “现场数据”

BUG 是表象，我们要发现内部原因需要更多的表象数据去推理，我们需要收集到足够的 “现场数据”。我们可以通过编程语言内置的输出语句 (比如：Go 的 `print`、`fmt.Printf` 等) 输出我们需要的信息；当然更为专业的方法是通过编程语言提供的专业的调试工具 (如：GDB) 设置断点来采集现场数据或重现 BUG。

- 定位问题所在

一旦有了 “现场数据”，接下来就需要你用 “火眼金睛” 从这一堆数据中找出你真正需要的；如果无法直接识别出真数据，那么可以根据数据做几组不同数据组合的模拟测试，在数据变化中 “去伪存真”，找到那个 “真悟空”。有了信赖的真实数据，你一般都可以根据代码逻辑推理出问题所在。但有些时候还是需要通过隔离代码、缩小嫌疑代码范围等方法才能锁定一些较难 BUG 的具体问题所在。

- 修正并验证

既然找到问题所在了，那剩下的工作就是修正它并重现验证；为验证问题已经修复而添加的新测试用例同时也补充了你的单元测试用例库。如果修正失败，那就从头开始新一轮分析过程。

最后，定期回顾你自己 “出产” 的 BUG 列表，你可能会发现很多 BUG 是你在预防阶段做的不够好而导致的，这样我们再回过头来思考如何在上一个阶段预防此类 “漏网之鱼”，这样就会形成一个良性循环。

## 2. Go 调试工具的选择

通过上面的描述，我们了解了调试的艰辛以及方法。接下来，我们就来聚焦于 Go 代码调试。“工欲善其事，必先利其器”，我们首先要进行的就是调试工具的选择。

在 Go 官方的 [2019 年 Go 开发者调查报告](https://blog.golang.org/survey2019-results)中，有关 Go 开发过程依赖的调试工具与技术的调查结果如下：

![33 掌握 Go 代码调试利器：delve](https://img-hello-world.oss-cn-beijing.aliyuncs.com/05c3da4d6ebb86b6eb9446345fbc94ac.png)
图 8-6-1：2019 年 Go 开发者调查报告有关调试的调查结果

从图中我们看到排名前两名的分别是：** 使用文本输出 (`fmt.Print` 等) 调试 Go 代码 ** 和 ** 使用专业调试器 (如：[Delve](https://github.com/go-delve/delve)、GDB) 在本机调试 Go 代码 **。这里我将采用第一种方法的称为 **“print 派”**，将第二种采用专门调试工具的称为 **“专业工具派”**。

通过编程语言内置的输出语句 (比如：Go 的 `fmt.Println`) 在代码特定位置输出特定信息辅助调试是目前最为常见的一种调试方法，不仅在 Go 语言中如此，在其他主流语言中亦是如此。就像[《Go Programming Blueprints》](https://book.douban.com/subject/26907929/)一书的作者[马特・莱尔 (Mat Ryer)](https://github.com/matrye) 所说的那样：“使用调试器进行调试比仅打印出值和思考要慢得多”。显然 “print 派” 的开发者更倾向于通过打印语句输出调试信息这种方式的**简单快捷、灵活直观且无需对外部有任何依赖**。

但开发界似乎也总有着一种 “偏见”：不使用专业调试器的程序员是不专业的。那么到底哪一派的观点更能站得住脚呢？笔者认为：print 语句辅助调试与采用专门调试工具对代码进行调试并不矛盾，它们之间是互相补充和相辅相成的。

“print 辅助调试” 更多用于在代码可修改的情况下的本地环境下的代码调试，通过 **“特定位置添加打印值” -> “编译执行” -> “根据输出结果调查思考” \**的调试循环来逐渐逼近真相。针对本地环境下的代码调试，专门的调试工具显然具有等价的功能，只是调试循环略繁琐：\**“启动调试器” -> “设置断点” -> “在调试器内运行程序” -> “断点处使用命令打印需要的信息” -> “单步调试等” -> “退出调试器”**。但专门调试工具可以运用在 “print 调试” 无法胜任的场景下，比如：

- 与 IDE 集成 (比如：[Goland](https://www.jetbrains.com/go/))，通过图形化操作可大幅简化专门调试工具的 “调试循环”，提供更佳的体验；
- 事后调查 (postmortem)
  - 调试 core dump 文件
  - 在生产环境通过挂接 (attach) 应用进程，深入应用进程内部进行调试

Go 早期版本 (Go 1.0 之前) 曾自带了一个名为 **OGLE** 的专用调试器 (绝大多数 gopher 都没有见识过该调试器，笔者也没有)，但该调试器并未随着 Go 正式发布，也就是说 Go 官方工具链中并未提供专门的调试器工具。Go 发行版中，除了标准的 Go 编译器 (gc) 之外，还有一个名为 [**gccgo**](https://tip.golang.org/doc/install/gccgo) 的编译器。和标准 Go 编译器 (gc) 相比，gccgo 具有如下特点：

- gccgo 是 GCC 编译器的新前端；
- Go 语言由 [Go 语言规范](https://tip.golang.org/ref/spec)定义和驱动演进的，gccgo 是另外一个实现了该语言规范的编译器，但与标准 Go 编译器实现的侧重点有不同；
- gccgo 编译速度较慢，但支持更为强大的优化能力；
- gccgo 复用了 GCC 后端，因此支持的处理器架构更多；
- gccgo 的演进速度与标准 Go 编译器 (gc) 的速度并不一致，按照最新[官方文档](https://tip.golang.org/doc/install/gccgo)，gcc8 等价于 go 1.10.1 的实现，而 gcc9 等价于 Go 1.12.2 的实现。

我们看到：通过 gccgo 编译而成的 Go 程序可以得到 GCC 成熟工具链集合的原生支持，包括使用强大的 GDB 进行调试。但由于 gccgo 不是主流，也不是我们重点考虑的内容，我们这里考虑的是基于标准 Go 编译器编译的代码的调试。

那么现成的 GDB 调试器是否可以调试通过标准 Go 编译器 (gc) 编译生成的 Go 程序呢？答案是肯定的。但 GDB 对标准 Go 编译器 (gc) 输出的程序的支持是不完善的，主要体现在 GDB 并不十分了解 Go 程序：

- Go 的栈管理、线程模型、运行时等与 GDB 所了解的执行模型有很大不同，这会导致 GDB 在调试过程中输出错误的结果，尤其是针对拥有大量并发的 Go 程序时，GDB 并不是一个可靠的调试器；
- 使用复杂，需加载插件 (`$GOROOT/src/runtime/runtime-gdb.py`) 才能更好地理解 Go 符号；
- GDB 无法理解一些 Go 类型信息、名称限定等，导致输出的栈信息和打印的变量类型信息难于识别、查看和分析；
- Go 1.11 后，编译后的可执行文件中调试信息默认是压缩的，低版本的 GDB 无法加载这些压缩的调试信息，除非显式使用 `go build -ldflags=-compressdwarf=false` 不执行调试信息压缩。

综上，GDB 显然也不是 Go 调试专门工具的最佳选择，虽然其适于调试带有 Cgo 代码的 Go 程序或事后调查调试。

[Delve](https://github.com/go-delve/delve) 是另一个 Go 语言调试器，该调试器工程于 2014 年由[德里克・帕克（Derek Parker）](https://github.com/derekparker)创建。Delve 旨在为 Go 提供一个简单的、功能齐全、易用使用和调用的调试工具。它紧跟 Go 语言版本演进，是目前 Go 调试器的**事实标准**。和 GDB 相比，Delve 的优势在于它可以更好地理解 Go 的一切，对并发程序有着很好支持，支持跨平台（至少是支持 Windows、MacOS、Linux 三大主流平台）并且它前后端分离的设计使得其可以非常容易地被集成到各种 IDE（如：Goland)、编译器插件 ([vscode go](https://github.com/Microsoft/vscode-go)、[vim-go](https://github.com/fatih/vim-go))、图形化调试器前端中 (如：[gdlv](https://github.com/aarzilli/gdlv))。接下来，我们就来看看如何使用 Delve 调试 Go 程序。

## 3. Delve 调试基础、原理与架构

### 1) 安装 delve

和所有 Go 工具一样，我们在任何平台上都可以通过下面命令安装 delve 的可执行程序 (delve 的目前最新版本为 1.4.1 版本)：

```shell
$go get github.com/go-delve/delve/cmd/dlv
go: downloading github.com/go-delve/delve v1.4.1
go: found github.com/go-delve/delve/cmd/dlv in github.com/go-delve/delve v1.4.1
go: downloading github.com/google/go-dap v0.2.0
go: downloading github.com/cpuguy83/go-md2man v1.0.10
go: downloading github.com/hashicorp/golang-lru v0.5.4 
```

安装成功后，可执行文件 `dlv` 将出现在 `$GOPATH/bin` 下面，确保你的环境变量 PATH 中包含该路径：

```shell
$dlv version
Delve Debugger
Version: 1.4.1
Build: $Id: bda606147ff48b58bde39e20b9e11378eaa4db46 
```

### 2) 使用 delve 调试 Go 代码示例

下面我们就来看一个使用 delve 调试 Go 代码的示例。示例代码 (`delve-demo1`) 本身十分简单，我们主要关注的是 delve 的调试循环。示例代码的目录结构如下：

```shell
// delve-demo1
$tree .                            
.
├── cmd
│   └── delve-demo1
│       └── main.go
├── go.mod
└── pkg
    └── foo
        └── foo.go

4 directories, 3 files 
```

我们列出示例中的主要代码：

```go
// delve-demo1/go.mod
module github.com/bigwhite/delve-demo1

go 1.14

// delve-demo1/cmd/delve-demo1/main.go
 1 package main
 2 
 3 import (
 4         "fmt"
 5 
 6         "github.com/bigwhite/delve-demo1/pkg/foo"
 7 )
 8 
 9 func main() {
10         a := 3
11         b := 10
12         c := foo.Foo(a, b)
13         fmt.Println(c)
14 }

// delve-demo1/pkg/foo/foo.go
1 package foo
2 
3 func Foo(step, count int) int {
4         sum := 0
5         for i := 0; i < count; i++ {
6                 sum += step
7         }
8         return sum
9 } 
```

我们在 `delve-demo1` 目录下可以通过 `dlv debug` 命令来直接调试 `delve-demo1` 的 `main` 包：

```shell
$cd delve-demo1 
$dlv debug github.com/bigwhite/delve-demo1/cmd/delve-demo1
Type 'help' for list of commands.
(dlv) 
```

注意：在 macOS 下，执行 `dlv debug` 前，我们需要首先赋予 dlv 使用系统调试 API 的权利：

```shell
sudo /usr/sbin/DevToolsSecurity -enable 
```

执行 dlv 后，dlv 会对被调试 Go 包进行编译并在当前工作目录下生成一个临时的二进制文件用于调试之用：

```shell
delve-demo1/__debug_bin* 
```

接下来，我们需要在待收集信息的 “现场” 前后设置断点，`delve` 提供了 `break` 命令来设置断点，可简写为 `b`：

```shell
(dlv) b main.go:12
Breakpoint 1 set at 0x10c2431 for main.main() ./main.go:12
(dlv) bp
Breakpoint runtime-fatal-throw at 0x1033970 for runtime.fatalthrow() $GOROOT/src/runtime/panic.go:1158 (0)
Breakpoint unrecovered-panic at 0x10339e0 for runtime.fatalpanic() $GOROOT/src/runtime/panic.go:1185 (0)
	print runtime.curg._panic.arg
Breakpoint 1 at 0x10c2431 for main.main() ./main.go:12 (0)
(dlv) 
```

我们在 `main.go` 源文件的第 12 行源码处设置了一个断点，通过 `breakpoints` 命令 (可简写为 `bp`) 可以查看已经设置的断点列表。不过注意上面这个断点没有名字，只有编号，是个匿名断点。如果我们要为断点起一个名字便于后续记忆、管理和使用，我们可以这样来做：

```go
(dlv) clear 1
Breakpoint 1 cleared at 0x10c2431 for main.main() ./main.go:12
(dlv) b b1 main.go:12
Breakpoint b1 set at 0x10c2431 for main.main() ./main.go:12
(dlv) bp
Breakpoint runtime-fatal-throw at 0x1033970 for runtime.fatalthrow() $GOROOT/src/runtime/panic.go:1158 (0)
Breakpoint unrecovered-panic at 0x10339e0 for runtime.fatalpanic() $GOROOT/src/runtime/panic.go:1185 (0)
	print runtime.curg._panic.arg
Breakpoint b1 at 0x10c2431 for main.main() ./main.go:12 (0)
(dlv) 
```

由于前面已经在 `main.go` 第 12 行设置了断点，我们先要清除掉这个断点 (使用 `clear` 命令)，然后再在相同位置 (`main.go` 的第 12 行) 设置一个具名断点：**b1**。

Delve 还支持设置**条件断点**。所谓条件断点，指的就是当满足某个条件时，被调试的目标程序才会在该断点处暂停。我们在 `foo.Foo` 中设置一个条件断点：

```go
(dlv) list foo.Foo
Showing chapter8/sources/delve-demo1/pkg/foo/foo.go:3 (PC: 0x10c2380)
   1:	package foo
   2:	
   3:	func Foo(step, count int) int {
   4:		sum := 0
   5:		for i := 0; i < count; i++ {
   6:			sum += step
   7:		}
   8:		return sum
(dlv) b b2 foo.go:6
Breakpoint b2 set at 0x10c23b8 for github.com/bigwhite/delve-demo1/pkg/foo.Foo() ./pkg/foo/foo.go:6
(dlv) cond b2 sum > 10 
```

设置完断点后，我们就可以运行程序了，我们可以通过 Delve 提供的 `continue` 命令执行程序 (可简写为 `c`)，执行后的程序会在下一个断点处停下来，如果没有断点，程序会一直执行下去，直到程序终止：

```go
(dlv) c
> [b1] main.main() ./cmd/delve-demo1/main.go:12 (hits goroutine(1):1 total:1) (PC: 0x10c2431)
     7:	)
     8:	
     9:	func main() {
    10:		a := 3
    11:		b := 10
=>  12:		c := foo.Foo(a, b)
    13:		fmt.Println(c)
    14:	}
(dlv) 
```

我们看到：通过 `continue` 执行程序后，调试页面停在了我们设置的第一个断点：`b1` 的位置，此时我们可以通过 Delve 提供的程序状态和内存查看命令收集一些 “现场数据”：

```go
(dlv) whatis a
int
(dlv) whatis b
int
(dlv) p a
3
(dlv) p b
10
(dlv) regs
   Rip = 0x00000000010c2431
   Rsp = 0x000000c0000a5ef8
   Rax = 0x000000c0000a5f78
   Rbx = 0x0000000000000000
   Rcx = 0x000000c000000180
   Rdx = 0x00000000010f9cc8
   Rsi = 0x0000000000000001
   Rdi = 0x000000c0000901d0
   Rbp = 0x000000c0000a5f78
    R8 = 0x000000000880480e
    R9 = 0x0000000000000011
   R10 = 0x00000000010a0dc0
   R11 = 0x0000000000000246
   R12 = 0x0000000000203000
   R13 = 0x0000000000000000
   R14 = 0x00000000000000c8
   R15 = 0x00000000000000d0
Rflags = 0x0000000000000212	[AF IF IOPL=0]
    Cs = 0x000000000000002b
    Fs = 0x0000000000000000
    Gs = 0x0000000000000000

(dlv) locals
a = 3
b = 10
c = 18390264
(dlv) x 0x10c2431
0x10c2431:   0x48
(dlv) 
```

常用的查看命令有如下几个：

- print (简写为 `p`)：输出源码中变量的值；
- whatis：输出后面的表达式的类型；
- regs：当前寄存器中的值；
- locals：当前函数栈本地变量列表（包括变量的值）；
- args：当前函数栈参数和返回值列表（包括参数和返回值的值）;
- examinemem（简写为 `x`)：查看某一内存地址上的值。

接下来，我们可以继续使用 `continue` 让程序继续执行到下一个断点或结束，亦可以使用 Delve 提供的 `next` 和 `step` 命令做单步调试。

使用 `next` 命令 (简写为 `n`) 可以让程序执行到断点处的下一行源码：

```go
 > [b1] main.main() ./cmd/delve-demo1/main.go:12 (hits goroutine(1):1 total:1) (PC: 0x10c2431)
     7:	)
     8:	
     9:	func main() {
    10:		a := 3
    11:		b := 10
=>  12:		c := foo.Foo(a, b)
    13:		fmt.Println(c)
    14:	}
(dlv) n
> main.main() ./cmd/delve-demo1/main.go:13 (PC: 0x10c2452)
     8:	
     9:	func main() {
    10:		a := 3
    11:		b := 10
    12:		c := foo.Foo(a, b)
=>  13:		fmt.Println(c)
    14:	}
(dlv) 
```

而使用 `step` 命令 (简写为 `s`) 也是单步调试，但如果断点处有函数调用，`step` 命令会进入断点所在行调用的函数：

```go
> [b1] main.main() ./cmd/delve-demo1/main.go:12 (hits goroutine(1):1 total:1) (PC: 0x10c2431)
     7:	)
     8:	
     9:	func main() {
    10:		a := 3
    11:		b := 10
=>  12:		c := foo.Foo(a, b)
    13:		fmt.Println(c)
    14:	}
(dlv) s
> github.com/bigwhite/delve-demo1/pkg/foo.Foo() ./pkg/foo/foo.go:3 (PC: 0x10c2380)
     1:	package foo
     2:	
=>   3:	func Foo(step, count int) int {
     4:		sum := 0
     5:		for i := 0; i < count; i++ {
     6:			sum += step
     7:		}
     8:		return sum
(dlv) 
```

如果此时我们再执行 `continue`，程序将会在断点 `b2` 处停下来：

```go
(dlv) c
> [b2] github.com/bigwhite/delve-demo1/pkg/foo.Foo() ./pkg/foo/foo.go:6 (hits goroutine(1):1 total:1) (PC: 0x10c23b8)
     1:	package foo
     2:	
     3:	func Foo(step, count int) int {
     4:		sum := 0
     5:		for i := 0; i < count; i++ {
=>   6:			sum += step
     7:		}
     8:		return sum
     9:	}
(dlv) locals
sum = 12
i = 4 
```

由于断点 `b2` 是一个条件断点，我们看到程序执行到满足断点条件：`num > 10` 才停了下来（此时 sum = 12)。

在此处我们通过 `stack` 命令 (简写为 `bt`) 命令可以输出调用栈信息：

```go
(dlv) bt
0  0x00000000010c23b8 in github.com/bigwhite/delve-demo1/pkg/foo.Foo
   at ./pkg/foo/foo.go:6
1  0x00000000010c2448 in main.main
   at ./cmd/delve-demo1/main.go:12
2  0x0000000001035e88 in runtime.main
   at $GOROOT/src/runtime/proc.go:203
3  0x0000000001063a31 in runtime.goexit
   at $GOROOT/src/runtime/asm_amd64.s:1373
(dlv) 
```

通过 `up` 和 `down` 命令，我们可以在函数调用栈的栈帧间进行跳转：

```go
(dlv) up
> [b2] github.com/bigwhite/delve-demo1/pkg/foo.Foo() ./pkg/foo/foo.go:6 (hits goroutine(1):1 total:1) (PC: 0x10c23b8)
Frame 1: ./cmd/delve-demo1/main.go:12 (PC: 10c2448)
     7:	)
     8:	
     9:	func main() {
    10:		a := 3
    11:		b := 10
=>  12:		c := foo.Foo(a, b)
    13:		fmt.Println(c)
    14:	}
(dlv) up
> [b2] github.com/bigwhite/delve-demo1/pkg/foo.Foo() ./pkg/foo/foo.go:6 (hits goroutine(1):1 total:1) (PC: 0x10c23b8)
Frame 2: $GOROOT/src/runtime/proc.go:203 (PC: 1035e88)
   198:			// A program compiled with -buildmode=c-archive or c-shared
   199:			// has a main, but it is not executed.
   200:			return
   201:		}
   202:		fn := main_main // make an indirect call, as the linker doesn't know the address of the main package when laying down the runtime
=> 203:		fn()
   204:		if raceenabled {
   205:			racefini()
   206:		}
   207:	
   208:		// Make racy client program work: if panicking on
(dlv) down
> [b2] github.com/bigwhite/delve-demo1/pkg/foo.Foo() ./pkg/foo/foo.go:6 (hits goroutine(1):1 total:1) (PC: 0x10c23b8)
Frame 1: ./cmd/delve-demo1/main.go:12 (PC: 10c2448)
     7:	)
     8:	
     9:	func main() {
    10:		a := 3
    11:		b := 10
=>  12:		c := foo.Foo(a, b)
    13:		fmt.Println(c)
    14:	}
(dlv) 
```

接下来，我们再执行几次 `continue` 直到 for 循环结束，程序将正常退出，我们也完成了一次调试循环：

```shell
(dlv) c
30
Process 69045 has exited with status 0 
```

如果要重启调试，我们无需退出 delve，只需执行 `restart`(简写为：`r`) 重启程序即可：

```shell
(dlv) r
Process restarted with PID 69081 
```

Delve 还支持在调试过程修改变量的值，并手工调用函数 (试验功能)，这样开发者就无需通过修改代码、重入调试来做某些验证了：

```go
(dlv) b b1 main.go:12             
Breakpoint b1 set at 0x10c2431 for main.main() ./cmd/delve-demo1/main.go:12
(dlv) c
> [b1] main.main() ./cmd/delve-demo1/main.go:12 (hits goroutine(1):1 total:1) (PC: 0x10c2431)
     7:	)
     8:	
     9:	func main() {
    10:		a := 3
    11:		b := 10
=>  12:		c := foo.Foo(a, b)
    13:		fmt.Println(c)
    14:	}
(dlv) set a = 4
(dlv) p a
4
(dlv) call foo.Foo(a, b)
> [b1] main.main() ./cmd/delve-demo1/main.go:12 (hits goroutine(1):2 total:2) (PC: 0x10c2431)
Values returned:
	~r2: 40

     7:	)
     8:	
     9:	func main() {
    10:		a := 3
    11:		b := 10
=>  12:		c := foo.Foo(a, b)
    13:		fmt.Println(c)
    14:	}
(dlv) 
```

我们看到在手工重新设置 `a = 40` 后，再手工调用 `call` 命令执行 `foo.Foo(a, b)` 后的结果为 `40`。

Delve 也可以通过直接调试源码文件的方式启动调试流程，这样的方式与调试包是等价的：

```shell
$dlv debug cmd/delve-demo1/main.go 
```

Delve 还可以通过 `exec` 子命令直接调试已经构建完的 Go 二进制程序文件，比如：

```go
$go build github.com/bigwhite/delve-demo1/cmd/delve-demo1
$dlv exec ./delve-demo1 
Type 'help' for list of commands.
(dlv) b b1 main.go:12
Breakpoint b1 set at 0x109ced1 for main.main() ./cmd/delve-demo1/main.go:12
(dlv) list main.go:12
Showing chapter8/sources/delve-demo1/cmd/delve-demo1/main.go:12 (PC: 0x109ced1)
   7:	)
   8:	
   9:	func main() {
  10:		a := 3
  11:		b := 10
  12:		c := foo.Foo(a, b)
  13:		fmt.Println(c)
  14:	}
(dlv) 
```

直接调试二进制文件时，Delve 会根据二进制文件中保存的源文件位置去到对应的路径下寻找对应的源文件并展示对应源码，如果我们把那个路径下的源文件挪走，那么再通过 `list` 命令展示源码就会出现错误：

```go
(dlv) list main.go:12
Showing chapter8/sources/delve-demo1/cmd/delve-demo1/main.go:12 (PC: 0x109ced1)
Command failed: open chapter8/sources/delve-demo1/cmd/delve-demo1/main.go: no such file or directory
(dlv) 
```

某些时候通过 Delve 直接调试构建后的二进制文件，可能会出现如下错误 (下面仅是模拟示例)：

```go
(dlv) break main.go:12
Command failed: could not find statement at chapter8/sources/delve-demo1/cmd/delve-demo1/main.go:12, please use a line with a statement 
```

明明 `main.go` 的第 12 行是一个函数调用，但 Delve 就是提示这行没有 Go 语句。这个问题的原因很可能是 Go 编译器对目标代码做了优化，比如将 `foo.Foo` 内联掉了。为了避免这样的问题，我们可以在编译的时候加入关闭优化的标志位，这样 Delve 就不会因目标代码优化而报出错误的信息了。

```shell
$go build -gcflags=all="-N -l" github.com/bigwhite/delve-demo1/cmd/delve-demo1 
```

#### 3) delve 架构与原理

为了便于各种调试器前端 (命令行、IDE、编辑器插件、图形化前端) 与 Delve 集成，Delve 采用了一个前后分离的架构：

![33 掌握 Go 代码调试利器：delve](https://img-hello-world.oss-cn-beijing.aliyuncs.com/348787b4a9ac30eb7afda4c6424b54f1.png)
图 8-6-2：Delve 分层架构

图中的 `UI Layer` 对应的就是我们使用的 `dlv` 命令行或 Goland/vim-go 中的调试器前端；`Service Layer` 显然用于前后端通信使用。Delve 真正施展的 “魔法” 是由下面的 `Symbolic Layer` 和 `Target Layer` 两层合作实现的。

`Target Layer` 通过各个操作系统提供的系统 API 接口来控制被调试目标进程，它并对被调试目标的源码没有任何了解，它实现的功能包括：

- 挂载 (attach)/ 分离 (detach) 目标进程
- 枚举目标进程中的线程
- 可以启动 / 停止单个线程（或整个进程）
- 接收和处理 “调试事件”（线程创建 / 退出以及线程在断点处暂停）
- 可以读写目标进程的内存
- 可以读写停止线程的 CPU 寄存器
- 可以读取 core dump 文件

真正了解被调试目标源码文件的是 `Symbolic Layer`，这一层通过读取 Go 编译器（包括链接器）以 [DWARF 格式 (一种标准的调试信息格式)](http://dwarfstd.org/) 写入到目标二进制文件中的调试符号信息来了解被调试目标源码，并实现了被调试目标进程中的地址、二进制文件中的调试符号以及源码相关信息三者之间的关系映射：

![33 掌握 Go 代码调试利器：delve](https://img-hello-world.oss-cn-beijing.aliyuncs.com/20bb81a085b2de953330aaab0597bbd4.png)
图 8-6-3：Delve Symbolic 层原理

## 4. 并发、Coredump 文件与挂接进程调试

### 1）delve 调试并发程序

Go 是原生支持并发的语言，并发的基本执行单元是 goroutine。前面提到过 GDB 对 Go 语言了解的不够，尤其是在 Go 程序由大量 goroutine 并发组成时。Delve 对并发 Go 程序有着良好的支持，通过 Delve 提供调试命令，我们可以在各个运行的 goroutine 间切换。下面我们来看一个调试并发程序的例子 delve-demo2。下面列出 delve-demo2 示例的部分代码，便于后面说明：

```go
// delve-demo2的目录结构
$tree .
.
├── cmd
│   └── delve-demo2
│       └── main.go
├── go.mod
└── pkg
    ├── bar
    │   └── bar.go
    └── foo
        └── foo.go

5 directories, 4 files

// delve-demo2/cmd/delve-demo2/main.go
... ...
func main() {
	var wg sync.WaitGroup
	wg.Add(1)
	go func() {
		for {
			d := 2
			e := 20
			f := bar.Bar(d, e)
			fmt.Println(f)
			time.Sleep(2 * time.Second)
		}
		wg.Done()
	}()
	a := 3
	b := 10
	c := foo.Foo(a, b)
	fmt.Println(c)
	wg.Wait()
	fmt.Println("program exit")
}

// delve-demo2/pkg/bar/bar.go
package bar

func Bar(step, count int) int {
	sum := 1
	for i := 0; i < count; i++ {
		sum *= step
	}
	return sum
} 
```

调试并发程序与普通程序没有两样，我们同样可通过 `dlv debug` 来调试对应包的方式启动调试：

```go
$dlv debug github.com/bigwhite/delve-demo2/cmd/delve-demo2
Type 'help' for list of commands.
(dlv) 
```

我们依然通过 `break` 命令设置断点，这次我们在新创建的 goroutine 中设置一个断点：

```go
(dlv) list main.go:19
Showing chapter8/sources/delve-demo2/cmd/delve-demo2/main.go:19 (PC: 0x10c2d95)
  14:		wg.Add(1)
  15:		go func() {
  16:			for {
  17:				d := 2
  18:				e := 20
  19:				f := bar.Bar(d, e)
  20:				fmt.Println(f)
  21:				time.Sleep(2 * time.Second)
  22:			}
  23:			wg.Done()
  24:		}()
(dlv) b b1 main.go:19
Breakpoint b1 set at 0x10c2d95 for main.main.func1() ./cmd/delve-demo2/main.go:19
(dlv) 
```

让 delve 执行目标程序：

```go
(dlv) c
30
> [b1] main.main.func1() ./cmd/delve-demo2/main.go:19 (hits goroutine(6):1 total:1) (PC: 0x10c2d95)
    14:		wg.Add(1)
    15:		go func() {
    16:			for {
    17:				d := 2
    18:				e := 20
=>  19:				f := bar.Bar(d, e)
    20:				fmt.Println(f)
    21:				time.Sleep(2 * time.Second)
    22:			}
    23:			wg.Done()
    24:		}()
(dlv) 
```

我们看到：main goroutine 输出了 `foo.Foo` 调用的返回结果 `30`，然后调试程序在 `main.go` 的第 19 行停了下来。这时我们通过 Delve 提供的 `goroutines` 命令来查看当前程序内的 goroutine 列表：

```go
(dlv) goroutines
  Goroutine 1 - User: $GOROOT/src/runtime/sema.go:56 sync.runtime_Semacquire (0x1045382)
  Goroutine 2 - User: $GOROOT/src/runtime/proc.go:305 runtime.gopark (0x103623b)
  Goroutine 3 - User: $GOROOT/src/runtime/proc.go:305 runtime.gopark (0x103623b)
  Goroutine 4 - User: $GOROOT/src/runtime/proc.go:305 runtime.gopark (0x103623b)
  Goroutine 5 - User: $GOROOT/src/runtime/proc.go:305 runtime.gopark (0x103623b)
* Goroutine 6 - User: ./cmd/delve-demo2/main.go:19 main.main.func1 (0x10c2d95) (thread 2076364)
[6 goroutines]
(dlv) 
```

我们看到当前被调试的目标程序共启动了 6 个 goroutine，Delve 已经将当前上下文自动切换到了断点 `b1` 所在的 goroutine 6，即前面有一个星号的那个 goroutine 中。接下来的命令操作都是在当前 goroutine 下执行的：

```go
(dlv) list
> [b1] main.main.func1() ./cmd/delve-demo2/main.go:19 (hits goroutine(6):1 total:1) (PC: 0x10c2d95)
    14:		wg.Add(1)
    15:		go func() {
    16:			for {
    17:				d := 2
    18:				e := 20
=>  19:				f := bar.Bar(d, e)
    20:				fmt.Println(f)
    21:				time.Sleep(2 * time.Second)
    22:			}
    23:			wg.Done()
    24:		}()
(dlv) n
> main.main.func1() ./cmd/delve-demo2/main.go:20 (PC: 0x10c2db6)
    15:		go func() {
    16:			for {
    17:				d := 2
    18:				e := 20
    19:				f := bar.Bar(d, e)
=>  20:				fmt.Println(f)
    21:				time.Sleep(2 * time.Second)
    22:			}
    23:			wg.Done()
    24:		}()
    25:		a := 3
(dlv) p f
1048576
(dlv) bt
0  0x00000000010c2db6 in main.main.func1
   at ./cmd/delve-demo2/main.go:20
1  0x0000000001063cc1 in runtime.goexit
   at /Users/tonybai/.bin/go1.14/src/runtime/asm_amd64.s:1373 
```

我们可以通过 `goroutine` 命令切换到其他 goroutine 中，比如切换到 main goroutine 中 (goroutine 1)：

```go
(dlv) goroutine 1
Switched from 6 to 1 (thread 2076364)
(dlv) bt
0  0x000000000103623b in runtime.gopark
   at /Users/tonybai/.bin/go1.14/src/runtime/proc.go:305
1  0x00000000010362f3 in runtime.goparkunlock
   at /Users/tonybai/.bin/go1.14/src/runtime/proc.go:310
2  0x000000000104570b in runtime.semacquire1
   at /Users/tonybai/.bin/go1.14/src/runtime/sema.go:144
3  0x0000000001045382 in sync.runtime_Semacquire
   at /Users/tonybai/.bin/go1.14/src/runtime/sema.go:56
4  0x000000000107cdc6 in sync.(*WaitGroup).Wait
   at /Users/tonybai/.bin/go1.14/src/sync/waitgroup.go:130
5  0x00000000010c2cb9 in main.main
   at ./cmd/delve-demo2/main.go:29
6  0x0000000001035e88 in runtime.main
   at /Users/tonybai/.bin/go1.14/src/runtime/proc.go:203
7  0x0000000001063cc1 in runtime.goexit
   at /Users/tonybai/.bin/go1.14/src/runtime/asm_amd64.s:1373
(dlv) 
```

我们看到：main goroutine 由于执行了 `sync.WaitGroup.Wait`，正处于挂起状态。

Delve 还提供了 `thread` 和 `threads` 命令，通过这两个命令我们可以查看当前启动的线程列表并在各个线程间切换，就像上面在 goroutine 间切换调试一样：

```go
(dlv) threads
  Thread 2075562 at :0
  Thread 2076362 at :0
  Thread 2076363 at :0
* Thread 2076364 at 0x10c2db6 ./cmd/delve-demo2/main.go:20 main.main.func1
  Thread 2076365 at :0
(dlv) 
```

我们看到：Delve 调试并发程序可以像调试普通程序一样简单方便。

### 2) 使用 delve 调试 core dump 文件

**core dump** 文件是当程序运行的过程中异常终止或崩溃时操作系统将程序当时的内存状态记录并保存生成的一个数据文件，该文件以 `core` 命名，也被称为**核心转储文件**。通过对操作系统记录的 core 文件中的数据的分析诊断，开发人员可以快速定位程序中存在的 Bug，这尤其适用于生产环境中的调试。根据 [Delve 官方文档](https://github.com/go-delve/delve/blob/master/Documentation/usage/dlv_core.md)，Delve 目前支持对 `linux/amd64`、`linux/arm64` 架构下产生的 core 文件的调试，以及 `Windows/amd64` 架构下产生的 minidump 小转储文件的调试。在这里我们以 `linux/amd64` 架构为例，看看如何使用 delve 调试 core dump 文件。

我们建立一个很明显会导致崩溃的 Go 程序：

```go
// delve-demo3/main.go
package main

import "fmt"

func main() {
	var p *int
	*p = 1
	fmt.Println("program exit")
} 
```

显然这个程序运行后将因为空指针解引用而崩溃。我们在 `Linux/amd64` 下 (ubuntu 18.04; go 1.14; delve 1.4.1) 进行这次调试。要想在 Linux 下让 Go 程序崩溃时产生 core 文件，我们需要做一些设置，因为默认情况下，Go 程序崩溃并不会产生 core 文件：

```go
$ulimit -c unlimited // 不限制core文件大小
$go build main.go
$GOTRACEBACK=crash ./main
panic: runtime error: invalid memory address or nil pointer dereference
[signal SIGSEGV: segmentation violation code=0x1 addr=0x0 pc=0x49142f]

goroutine 1 [running]:
panic(0x4a6ee0, 0x55c880)
	/root/.bin/go1.14/src/runtime/panic.go:1060 +0x420 fp=0xc000068ef8 sp=0xc000068e50 pc=0x42e7f0
runtime.panicmem(...)
	/root/.bin/go1.14/src/runtime/panic.go:212
runtime.sigpanic()
	/root/.bin/go1.14/src/runtime/signal_unix.go:687 +0x3da fp=0xc000068f28 sp=0xc000068ef8 pc=0x4429ea
main.main()
	/root/test/go/delve/main.go:8 +0x1f fp=0xc000068f88 sp=0xc000068f28 pc=0x49142f
runtime.main()
	/root/.bin/go1.14/src/runtime/proc.go:203 +0x212 fp=0xc000068fe0 sp=0xc000068f88 pc=0x431222
runtime.goexit()
	/root/.bin/go1.14/src/runtime/asm_amd64.s:1373 +0x1 fp=0xc000068fe8 sp=0xc000068fe0 pc=0x45b911
... ...
goroutine 5 [runnable]:
runtime.runfinq()
	/root/.bin/go1.14/src/runtime/mfinal.go:161 fp=0xc0000307e0 sp=0xc0000307d8 pc=0x414f60
runtime.goexit()
	/root/.bin/go1.14/src/runtime/asm_amd64.s:1373 +0x1 fp=0xc0000307e8 sp=0xc0000307e0 pc=0x45b911
created by runtime.createfing
	/root/.bin/go1.14/src/runtime/mfinal.go:156 +0x61
Aborted (core dumped) 
```

我们看到程序如预期崩溃并输出 core dumped，在当前目录下，我们能看到 core 文件已经产生了并且其体积不小 (101M)：

```shell
$ls -lh
total 103M
-rw------- 1 root root 101M May 28 14:55 core
-rwxr-xr-x 1 root root 2.0M May 28 14:55 main
-rw-r--r-- 1 root root  102 May 28 14:54 main.go 
```

接下来就轮到 Delve 登场了，我们假装并不知道问题出在哪里：)。我们使用 `dlv core` 命令对产生的 core 文件进行调试：

```go
$dlv core ./main ./core
Type 'help' for list of commands.
(dlv) bt
 0  0x000000000045d4a1 in runtime.raise
    at /root/.bin/go1.14/src/runtime/sys_linux_amd64.s:165
 1  0x0000000000442acb in runtime.dieFromSignal
    at /root/.bin/go1.14/src/runtime/signal_unix.go:721
 2  0x0000000000442f5e in runtime.sigfwdgo
    at /root/.bin/go1.14/src/runtime/signal_unix.go:935
 3  0x00000000004419d4 in runtime.sigtrampgo
    at /root/.bin/go1.14/src/runtime/signal_unix.go:404
 4  0x000000000045d803 in runtime.sigtramp
    at /root/.bin/go1.14/src/runtime/sys_linux_amd64.s:389
 5  0x000000000045d8f0 in runtime.sigreturn
    at /root/.bin/go1.14/src/runtime/sys_linux_amd64.s:481
 6  0x0000000000442c5a in runtime.crash
    at /root/.bin/go1.14/src/runtime/signal_unix.go:813
 7  0x000000000042ee54 in runtime.fatalpanic
    at /root/.bin/go1.14/src/runtime/panic.go:1212
 8  0x000000000042e7f0 in runtime.gopanic
    at /root/.bin/go1.14/src/runtime/panic.go:1060
 9  0x00000000004429ea in runtime.panicmem
    at /root/.bin/go1.14/src/runtime/panic.go:212
10  0x00000000004429ea in runtime.sigpanic
    at /root/.bin/go1.14/src/runtime/signal_unix.go:687
11  0x000000000049142f in main.main
    at ./main.go:8
12  0x0000000000431222 in runtime.main
    at /root/.bin/go1.14/src/runtime/proc.go:203
13  0x000000000045b911 in runtime.goexit
    at /root/.bin/go1.14/src/runtime/asm_amd64.s:1373
(dlv) 
```

我们看到通过 `stack`(简写为 `bt`) 命令输出的函数调用栈多为 Go 运行时的函数，我们唯一熟悉的就是 `main.main`，于是，我们通过 `frame` 命令跳到 `main.main` 这个函数栈帧中：

```go
(dlv) frame 11
> runtime.raise() /root/.bin/go1.14/src/runtime/sys_linux_amd64.s:165 (PC: 0x45d4a1)
Warning: debugging optimized function
Frame 11: ./main.go:8 (PC: 49142f)
     3:	import "fmt"
     4:	
     5:	func main() {
     6:		var p *int
     7:		p = nil
=>   8:		*p = 1
     9:		fmt.Println("program exit")
    10:	}
(dlv) 
```

因为代码简单，这里我们一眼就能看出 `=>` 所指的那一行代码存在的问题。如果代码复杂且涉及函数调用较多，我们还可以继续通过 `up` 和 `down` 在各层函数栈帧中搜寻问题的真因。

### 3) 使用 delve 挂接 (attach) 到正在运行的进程进行调试

有些特定情况下，我们可能需要对正在运行的 Go 应用进程进行调试。不过这类调试是有较大风险的：调试器一旦成功挂接到正在运行的进程中，调试器就掌握了进程执行的指挥权，并且挂接后，正在运行的 goroutine 都会暂停下来，等待调试器的进一步指令。因此，在生产环境不到万不得已，请不要使用这种调试方法。

我们以 delve-demo2 为例简要说明一下通过 delve 挂接到进程进行调试的方法。我们首先编译并运行 delve-demo2：

```shell
$cd delve-demo2
$go build github.com/bigwhite/delve-demo2/cmd/delve-demo2
$./delve-demo2
30
1048576
1048576
... ... 
```

接下来，我们使用 `delve attach` 命令来切入 `delve-demo2` 应用正在运行的进程：

```shell
$ps -ef|grep delve-demo2
  501 75863 63197   0  3:33下午 ttys011    0:00.02 ./delve-demo2

$dlv attach 75863 ./delve-demo2 
Type 'help' for list of commands.
(dlv) 
```

delve 一旦成功切入 delve-demo2 进程，delve-demo2 进程内的所有 goroutine 都将暂停运行，等待 delve 的进一步指令。现在我们可以利用之前使用过的所有命令操作被调试的进程了。比如：查看看 goroutine 列表、设置断点、在断点处暂停调试等：

```go
(dlv) goroutines
  Goroutine 1 - User: $GOROOT/src/runtime/sema.go:56 sync.runtime_Semacquire (0x103f472)
  Goroutine 2 - User: $GOROOT/src/runtime/proc.go:305 runtime.gopark (0x1030f60)
  Goroutine 3 - User: $GOROOT/src/runtime/proc.go:305 runtime.gopark (0x1030f60)
  Goroutine 4 - User: $GOROOT/src/runtime/proc.go:305 runtime.gopark (0x1030f60)
  Goroutine 17 - User: $GOROOT/src/runtime/proc.go:305 runtime.gopark (0x1030f60)
  Goroutine 18 - User: $GOROOT/src/runtime/time.go:198 time.Sleep (0x104ba7a)
[6 goroutines]
(dlv) b b1 main.go:19
Breakpoint b1 set at 0x109d511 for main.main.func1() ./cmd/delve-demo2/main.go:19
(dlv) c
> [b1] main.main.func1() ./cmd/delve-demo2/main.go:19 (hits goroutine(18):1 total:1) (PC: 0x109d511)
Warning: debugging optimized function
    14:		wg.Add(1)
    15:		go func() {
    16:			for {
    17:				d := 2
    18:				e := 20
=>  19:				f := bar.Bar(d, e)
    20:				fmt.Println(f)
    21:				time.Sleep(2 * time.Second)
    22:			}
    23:			wg.Done()
    24:		}()
(dlv) 
```

被调试的进程将在调试器的 “指挥” 下执行并在代码执行到 I/O 输出时向标准输出输出对应的变量值。

实例：

```go
 2022-01-21 14:04:01 ⌚  ubuntu in ~/workspace/50bestpratices
○ → tree -L 3 ./
./
├── example
│   ├── sources
│   │   ├── go-tools
│   │   │   ├── build_with_race_option.go
│   │   │   └── build_with_race_option_test.go
│   │   ├── go-trap
│   │   └── sources.go
│   ├── string.go
│   └── string_test.go
├── go.mod
├── go.sum
└── main.go

4 directories, 44 files

 2022-01-21 14:08:14 ⌚  ubuntu in ~/workspace/50bestpratices
○ → dlv debug .
Type 'help' for list of commands.
(dlv) bp
Breakpoint runtime-fatal-throw (enabled) at 0x4389c0 for runtime.throw() /home/going/go/go1.17.5/src/runtime/panic.go:1188 (0)
Breakpoint unrecovered-panic (enabled) at 0x438d20 for runtime.fatalpanic() /home/going/go/go1.17.5/src/runtime/panic.go:1271 (0)
        print runtime.curg._panic.arg
(dlv) b b1 ./example/sources/go-tools/build_with_race_option.go:13
Breakpoint b1 set at 0x6e501c for github.com/jxs1211/example/sources/go-tools.Println() ./example/sources/go-tools/build_with_race_option.go:13
(dlv) c
2022-01-21T14:08:27-08:00 error layer=debugger error loading binary "/lib/x86_64-linux-gnu/libpthread.so.0": could not parse .eh_frame section: unknown CIE_id 0x9daad7e4 at 0x0
> [b1] github.com/jxs1211/example/sources/go-tools.Println() ./example/sources/go-tools/build_with_race_option.go:13 (hits goroutine(1):1 total:1) (PC: 0x6e501c)
     8:         "time"
     9: )
    10:
    11: func Println() string {
    12:         var a *int
=>  13:         *a = 1
    14:         // fmt.Println(a)
    15:         return "go-tools: Println"
    16: }
    17:
    18: func ConcurrentlyWriteMap() {
(dlv) bt
0  0x00000000006e501c in github.com/jxs1211/example/sources/go-tools.Println
   at ./example/sources/go-tools/build_with_race_option.go:13
1  0x00000000006e507f in github.com/jxs1211/example/sources.Println
   at ./example/sources/sources.go:6
2  0x00000000006e51d9 in github.com/jxs1211/example.PprofStandalone4
   at ./example/pprof_standalone4.go:20
3  0x00000000006e5697 in main.main
   at ./main.go:562
4  0x000000000043af93 in runtime.main
   at /home/going/go/go1.17.5/src/runtime/proc.go:255
5  0x0000000000468fe1 in runtime.goexit
   at /home/going/go/go1.17.5/src/runtime/asm_amd64.s:1581
(dlv) c
> [unrecovered-panic] runtime.fatalpanic() /home/going/go/go1.17.5/src/runtime/panic.go:1271 (hits goroutine(1):1 total:1) (PC: 0x438d20)
Warning: debugging optimized function
        runtime.curg._panic.arg: interface {}(string) "runtime error: invalid memory address or nil pointer dereference"
  1266: // fatalpanic implements an unrecoverable panic. It is like fatalthrow, except
  1267: // that if msgs != nil, fatalpanic also prints panic messages and decrements
  1268: // runningPanicDefers once main is blocked from exiting.
  1269: //
  1270: //go:nosplit
=>1271: func fatalpanic(msgs *_panic) {
  1272:         pc := getcallerpc()
  1273:         sp := getcallersp()
  1274:         gp := getg()
  1275:         var docrash bool
  1276:         // Switch to the system stack to avoid any stack growth, which
(dlv) c
panic: runtime error: invalid memory address or nil pointer dereference
[signal SIGSEGV: segmentation violation code=0x1 addr=0x0 pc=0x6e501e]

goroutine 1 [running]:
github.com/jxs1211/example/sources/go-tools.Println()
        /home/going/workspace/50bestpratices/example/sources/go-tools/build_with_race_option.go:13 +0x1e
github.com/jxs1211/example/sources.Println()
        /home/going/workspace/50bestpratices/example/sources/sources.go:6 +0x1f
github.com/jxs1211/example.PprofStandalone4()
        /home/going/workspace/50bestpratices/example/pprof_standalone4.go:20 +0x39
main.main()
        /home/going/workspace/50bestpratices/main.go:562 +0x17
Process 119975 has exited with status 2
(dlv) c
Process 119975 has exited with status 2
(dlv) 
```



## 5. 小结

Delve 的功能还不限于上面的这些调试场景，比如：delve 支持调试单元测试代码 (`delve test`) 等。鉴于篇幅有限，这里不能一一细说。另外 Delve 的单步调试使其非常适合做源码分析辅助工具，在 Delve 这柄 “放大镜” 面前，再深奥复杂的源码流程也会被看得一清二楚。

本节要点：

- 通过编译器、静态代码检查工具 (linter)、编写单元测试等最佳实践尽量降低调试在整个开发过程中的比例；
- 通过编程语言内置的 print 语句辅助调试与采用专门的调试器调试代码是相辅相成的；
- 专门的调试器适用于与外部调试前端集成 (编辑器插件、IDE、其他图形化前端）；
- 专门的调试器适用于对 Core Dump 文件的调试分析以及生产环境的挂接进程调试；
- 相比于 GDB，Delve 能更好的理解 Go 程序，并支持对 Go 并发程序的调试、Core 文件调试、在线挂接进程调试以及对单元测试的调试。