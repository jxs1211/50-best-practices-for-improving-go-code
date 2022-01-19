28 一文告诉你测试包的包名要不要带“\_test”后缀

## 一文告诉你测试包的包名要不要带“\_test”后缀

**Go 原生在工具链和标准库中提供对测试的支持**，这算是 Go 语言在工程实践方面一个创新，也是 Go 相较于其他主流语言的一个突出亮点之一。

在 Go 中我们针对**包（package）** 编写测试代码。测试代码与包代码放在同一个包目录下，并且 Go 要求所有测试代码都存放在以`*_test.go`结尾的文件中。这使得 Go 开发人员可以一眼就分辨出哪些文件存放的是包代码，哪些文件存放的是针对该包的测试代码。

`go test`命令也是通过同样的方式将包代码和包测试代码区分开的。 `go test`将所有包目录下的`*_test.go`文件编译成一个临时二进制文件(我们可以通过`go test -c`显式编译出该文件)，并执行该文件，后者将执行各个测试源文件中的名字格式为`TestXxx`函数所代表的测试用例并输出测试执行结果。

## 1. 官方文档的“自相矛盾”

Go 原生支持测试的两大要素：`go test`命令和`testing`包是 Gopher 们学习 Go 代码测试的必经之路。

下面是关于`testing`包的一段[官方文档（基于 Go 1.14）摘录](https://tip.golang.org/pkg/testing/)：

> 要编写一个新的测试集（test suite），创建一个包含`TestXxx`函数的以`_test.go`为文件名结尾的文件。**将这个测试文件放在与被测试包相同的包下面**。编译被测试包时，该文件将被排除在外；执行`go test`时，该文件将被包含在内。

同样是官方文档，在介绍`go test`命令行工具时，[文档](https://tip.golang.org/cmd/go/#hdr-Test_packages)如是说：

> 那些包名中**带有`_test`后缀的测试文件**将被编译成一个独立的包，这个包之后会被链接到主测试二进制文件中并运行。

对比这两段官方文档，我们发现了一处[“自相矛盾”](https://github.com/golang/go/issues/25223)的地方：`testing`包文档告诉我们将测试代码放入**与被测试包同名的包中**；而`go test`命令行帮助文档则提到会将**包名中带有`_test`后缀的测试文件**编译成一个独立的包。

我们用一个例子来直观说明一下这个“矛盾”：如果我们要测试的包为`foo`，`testing`包的帮助文档告诉我们把对 foo 包的测试代码**放在包名为`foo`的测试文件**中；而`go test`命令行帮助文档则告诉我们把 foo 包的测试代码**放在包名为`foo_test`的测试文件**中。

我们将测试代码放在与被测包名相同的包下面的测试方法称为 **“包内测试”**，我们可以通过下面命令查看哪些测试源文件使用了“包内测试”：

```shell
$go list -f={{.TestGoFiles}} . 
```

我们将另外一种将测试代码放在“`被测包包名_test`”的包下面的测试方法称为 **“包外测试”**。同样，我们也可以通过下面命令查看哪些测试源文件使用了“包外测试”：

```shell
$go list -f={{.XTestGoFiles}} . 
```

那么我们究竟是选择**包内测试**还是**包外测试**呢？在给出结论之前，我们将分别对这两种方法做一个详细分析。

## 2. 包内测试 vs. 包外测试

### 1) Go 标准库中包内测试和包外测试的使用情况

Go 标准库是 Go 代码风格和惯用法一贯的风向标。我们先来看看标准库中“包内测试”和“包外测试”各自的比重：

在`$GOROOT/src`目录下（Go 1.14 版本），执行下面命令组合：

```shell
// 统计标准库中采用包内测试的测试文件数量
$find . -name "*_test.go" |xargs grep package |grep ':package'|grep -v "_test$"|wc -l    
     691                   

// 统计标准库中采用包外测试的测试文件数量
$find . -name "*_test.go" |xargs grep package |grep ':package'|grep "_test$"|wc -l
     448 
```

这并非是一个十分精确的统计，但一定程度上却能说明：包内测试和包外测试似乎各有各的优势。我们再以`net/http`这个被广泛使用的明星级别的包为例，看看包内测试和包外测试在该包测试中的应用：

```shell
进入$GOROOT/src/net/http目录下，分别执行下面命令：

$go list -f={{.XTestGoFiles}}      
[alpn_test.go client_test.go clientserver_test.go example_filesystem_test.go example_handle_test.go example_test.go fs_test.go main_test.go request_test.go serve_test.go sniff_test.go transport_test.go]

$go list -f={{.TestGoFiles}}  
[cookie_test.go export_test.go filetransport_test.go header_test.go http_test.go proxy_test.go range_test.go readrequest_test.go requestwrite_test.go response_test.go responsewrite_test.go server_test.go transfer_test.go transport_internal_test.go] 
```

我们看到，在针对`net/http`的测试代码中，包内测试和包外测试的使用仍然不分伯仲。

### 2) 包内测试的优势与不足

由于 Go 构建工具链在编译包时会自动根据文件名是否具有`_test.go`后缀将包源文件和包的测试源文件分开，测试代码不会进入包正常构建的范畴，因此测试代码使用与被测包名相同的**包内测试**方法是一个很自然的选择。

包内测试这种方法本质上是一种**“白盒测试”**方法。由于测试代码与被测包源码在同一包名下，测试代码**可以访问该包下的所有符号**，无论是导出符号还是未导出符号；并且由于包的内部实现逻辑对测试代码是透明的，包内测试**可以更为直接地构造测试数据和实施测试逻辑**，并且可以**很容易达到**较高的 **[测试覆盖率](https://blog.golang.org/cover)** 。因此对于追求高测试覆盖率的项目而言，包内测试是不二之选。

但在实践中，实施包内测试也经常会遇到如下的问题。

- 测试代码自身需要经常性的维护

包内测试的“白盒测试”的本质意味着它是一种**面向实现的测试**。测试代码的测试数据构造和测试逻辑通常与被测包的特定数据结构设计和函数/方法的具体实现逻辑是**紧耦合**的。这样一旦被测包的数据结构设计出现调整或函数/方法的实现逻辑出现变动，那么对应的测试代码也要随之同步调整，否则整个包的将无法通过测试甚至测试代码本身的构建都会失败。而包的内部实现逻辑又是易变的，其优化调整是一种经常性行为，这就意味着采用包内测试的测试代码也需要经常性的维护。

- 硬伤-“包循环引用”

采用包内测试可能会遇到一个绕不过去的硬伤-“包循环引用”，我们看下面示意图：

![28 一文告诉你测试包的包名要不要带“\_test”后缀](https://img-hello-world.oss-cn-beijing.aliyuncs.com/a30499d5e9ce263cb78e5a2386c76f7a.png)

图8-1-1: 包内测试的“包循环引用”

在图中我们看到：对**包 c**进行测试的代码(`c_test.go`)采用了包内测试的方法，其测试代码位于**包 c**下面，测试代码导入并引用了**包 d**，而**包 d**本身却导入引用了**包 c**，这种包循环引用是 Go 编译器所不允许的。

如果 Go 标准库对`strings`包的测试采用包内测试会遭遇什么呢

![28 一文告诉你测试包的包名要不要带“\_test”后缀](https://img-hello-world.oss-cn-beijing.aliyuncs.com/864fc37a94e8e3f4c0fb5a7e5326d664.png)

图8-1-2: 对标准库strings进行包内测试将遭遇“包循环引用”

从上图中我们看到 Go 测试代码必须要导入引用的`testing`包引用了`strings`包，这样如果 strings 包仍然使用包内测试方法，就必然会在测试代码中出现`strings`包与`testing`包循环引用的情况。于是当我们在标准库`string`包目录下执行下面命令时，我们得到：

```powershell
// 在$GOROOT/src/strings目录下
$go list -f {{.TestGoFiles}} .
[export_test.go] 
```

我们看到标准库`strings`包并未采用包内测试的方法(**注**：`export_test.go`并非包内测试的测试源文件，这个后续会有详细说明)。

### 3) 包外测试-仅针对导出 API 的测试

因为“包循环引用”的事实存在，Go 标准库无法针对`strings`包实施包内测试，而解决这一问题的自然就是**包外测试**了：

```shell
// 在$GOROOT/src/strings目录下
$go list -f {{.XTestGoFiles}} .
[builder_test.go compare_test.go example_test.go reader_test.go replace_test.go search_test.go strings_test.go] 
```



与包内测试本质是**面向实现的白盒测试**不同的是，包外测试本质则是一种**面向接口的黑盒测试**。这里的“接口”指的就是被测试包对外导出的 API，这些 API 是被测包与外部交互的契约，契约一旦确定就会长期保持稳定，无论被测包内部实现逻辑、数据结构设计如何调整优化，一般都不会影响这些“契约”。这一本质让包外测试代码与被测试包充分解耦，从而使得针对这些导出 API 进行测试的包外测试代码**表现出十分“健壮”的特性** - 即很少随着被测代码内部实现逻辑的调整而进行调整和维护。

包外测试将测试代码放入不同于被测试包的独立包的同时，也使得包外测试不再像包内测试那样存在“包循环引用”的硬伤，我们还以标准库中的`strings`包为例：

![28 一文告诉你测试包的包名要不要带“\_test”后缀](https://img-hello-world.oss-cn-beijing.aliyuncs.com/4a7c60b3b888ef4e2330db812d664413.png)

图8-1-3: 标准库strings包采用包外测试后解决了“包循环引用”问题

从图中我们看到，采用包外测试的 strings 包将测试代码放入`strings_test`包下面，`strings_test`包既引用了被测试包`strings`，又引用了`testing`包，这样一来原先采用包内测试的`strings`包与`testing`包的循环引用被轻易地“解”开了。

包外测试这种纯黑盒的测试还有一个功能域之外的好处，那就是可以更加聚焦地从用户视角验证被测试包导出 API 的设计的合理性和易用性。

不过包外测试的不足也是显而易见的，那就是存在 **“测试盲区”**。由于测试代码与被测试目标并不在同一包名下，测试代码仅有权访问被测包的导出符号，并且仅能通过导出 API 这一有限的“窗口”并结合构造特定数据来验证被测包行为。在这样的约束下，很容易出现 **对被测试包的测试覆盖不足** 的情况。

Go 标准库的实现者们为我们提供了一个解决包外测试这个问题的惯用法：**安插“后门”**。这个后门就是前面曾提到过的`export_test.go`文件。该文件中的代码位于被测包名下，但它既不会被包含在正式产品代码中（因为位于_test.go 文件中），又不包含任何测试代码，它仅用于将被测包的内部符号在测试阶段暴露给包外测试代码：

```go
// $GOROOT/src/fmt/export_test.go
package fmt

var IsSpace = isSpace
var Parsenum = parsenum 
```

the backdoor's function was used in external package test

```shell
2022-01-19 01:51:17 ⌚  ubuntu in ~/workspace/50bestpratices
○ → go list -f {{.XTestGoFiles}} /home/going/go/go1.17.5/src/fmt/
[errors_test.go example_test.go fmt_test.go gostringer_example_test.go scan_test.go stringer_example_test.go stringer_test.go]

 2022-01-19 01:54:40 ⌚  ubuntu in ~/workspace/50bestpratices
○ → grep IsSpace /home/going/go/go1.17.5/src/fmt/fmt_test.go 
func TestIsSpace(t *testing.T) {
        // IsSpace = isSpace is defined in export_test.go.
                if IsSpace(i) != unicode.IsSpace(i) {
                        t.Errorf("isSpace(%U) = %v, want %v", i, IsSpace(i), unicode.IsSpace(i))
```

或者是定义一些辅助包外测试的代码，比如扩展被测包的方法集合：

```go
// $GOROOT/src/strings/export_test.go
package strings

func (r *Replacer) Replacer() interface{} {
        r.once.Do(r.buildOnce)
        return r.r
}

func (r *Replacer) PrintTrie() string {
        r.once.Do(r.buildOnce)
        gen := r.r.(*genericReplacer)
        return gen.printNode(&gen.root, 0)
}
... ... 
```

我们可以用一幅图来直观展示`export_test.go`这个“后门”在不同阶段的角色(以`fmt`包为例)：

![28 一文告诉你测试包的包名要不要带“\_test”后缀](https://img-hello-world.oss-cn-beijing.aliyuncs.com/d0f6959501306820a1c69ed0201e377b.png)

图8-1-4: export_test.go为包外测试充当“后门”

通过上图，我们可以看到：`export_test.go`仅在`go test`阶段与被测试包（`fmt`）一并被构建入最终的测试二进制文件中。在这个过程中，包外测试代码（`fmt_test`）可以通过导入 **被测试包（`fmt`）** 来访问`export_test.go`中的导出符号（比如：`IsSpace`或对`fmt`包的扩展）。而`export_test.go`相当于在测试阶段扩展了包外测试代码的视野，让很多本来很难覆盖到的测试路径变得容易了，进而可以让包外测试可以覆盖更多被测试包中的执行路径。

```go
package fmt_test

import (
        "bytes"
        . "fmt"
        ...
)
```



### 4) 优先使用包外测试

经过上面的比较，我们发现包内测试与包外测试各有自己的优点与不足，那么在 Go 测试编码实践中我们究竟该选择哪种测试方式呢？关于这个问题，目前并无标准答案。以笔者的认知，基于实际中开发人员对编写测试代码的热情和投入时间，我个人更倾向于优先选择包外测试，理由如下。包外测试可以：

- 优先保证被测试包导出 API 的正确性；
- 可从用户角度验证导出 API 的有效性；
- 保持测试代码的健壮性，尽可能地降低对测试代码维护的投入；
- 不失灵活！可通过`export_test.go`这个“后门”来导出我们需要的内部符号，满足对包内实现逻辑窥探的需求。

当然`go test`也完全支持对被测包同时运用`包内测试`和`包外测试`两种测试方法，就像标准库`net/http`包那样。在这种情况下，`包外测试`由于将测试代码放入独立的包中，它更适合编写偏向**集成测试**的用例，它可以任意导入外部包，并测试与外部多个组件的交互。比如：`net/http`包的`serve_test.go`中就利用`httptest包`构建的模拟 Server 来测试相关接口；而包内测试更聚焦于内部逻辑的测试，通过给函数/方法传入一些特意构造的数据的方式来验证内部逻辑的正确性，比如：`net/http`包的`response_test.go`。

我们还可以通过测试代码的文件名来区分所属测试类别，比如：`net/http`包就使用`transport_internal_test.go`这个名字来明确指明该测试文件采用包内测试的方法。而对应的`transport_test.go`则是一个采用包外测试的源文件。

## 4. 小结

本节要点：

- go test 执行测试的原理；
- 理解包内测试的优点与不足；
- 理解包外测试的优点与不足；
- 掌握通过`export_test.go`为包外测试添加“后门”的惯用法；
- 优先使用包外测试；
- 当运用包外测试与包内测试共存的方式时，可考虑让包外测试和包内测试聚焦于不同测试类别。