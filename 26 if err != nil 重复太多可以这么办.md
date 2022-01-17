26 if err != nil 重复太多可以这么办

## if err != nil 重复太多可以这么办

## 1. 两种观点

在上一节谈 Go 错误处理的策略时我们就提到过：与 C++、C#、Java 和 Python 等许多支持异常处理的主流编程语言所采用的**对隐式结果的隐式错误检查**不同，Go 在最初设计时就有意识地**选择了使用显式错误结果和显式的错误检查**。

但 Go 在错误处理方面体现出的这种与如今主流语言的格格不入，让很多来自这些主流语言的 Go 初学者感到困惑：Go 代码中反复出现了太多的且方法单一的错误检查`if err != nil`。比如下面这段摘自[“Go2 错误处理概述”](https://github.com/golang/proposal/blob/master/design/go2draft-error-handling-overview.md)中的代码：

```go
func CopyFile(src, dst string) error {
	r, err := os.Open(src)
	if err != nil {
		return fmt.Errorf("copy %s %s: %v", src, dst, err)
	}
	defer r.Close()

	w, err := os.Create(dst)
	if err != nil {
		return fmt.Errorf("copy %s %s: %v", src, dst, err)
	}

	if _, err := io.Copy(w, r); err != nil {
		w.Close()
		os.Remove(dst)
		return fmt.Errorf("copy %s %s: %v", src, dst, err)
	}

	if err := w.Close(); err != nil {
		os.Remove(dst)
		return fmt.Errorf("copy %s %s: %v", src, dst, err)
	}
} 
```

对 Go 错误处理的“诟病”也体现在了 Go 官方用户调查数据当中。下图来自 2018 年[Go 官方用户调查的结果](https://blog.golang.org/survey2018-results)：

![26 if err != nil 重复太多可以这么办](https://img-hello-world.oss-cn-beijing.aliyuncs.com/4fad948c4206402feba53f8ab650d54c.png)

图7-2-1: 2018年Go官方用户调查的结果节选

我们看到在**“你使用 Go 面临的最大的挑战”**这一项调查中，参与调查的用户将“错误处理(error handling)”列在了**第五位**。在之前[2016 年](https://blog.golang.org/survey2016-results)和[2017 年](https://blog.golang.org/survey2017-results)两次官方用户调查中，错误处理也名列前茅。这也促使 Go 团队在**Go2**的演进和开发计划中，将 Go 错误处理的改善列为了一项重点工作。下面是 Go 团队在错误处理改善方面的工作梳理：

- 2018 年 8 月
  - [Go2 错误处理(error handling)设计草案](https://github.com/golang/proposal/blob/master/design/go2draft-error-handling.md)
  - [Go2 错误检查(error inspecting)设计草案](https://github.com/golang/proposal/blob/master/design/go2draft-error-inspection.md)
- 2019 年 6 月
  - [原生 Go 错误检查-try 设计草案](https://github.com/golang/proposal/blob/master/design/32437-try-builtin.md)
  - [tryhard 项目](https://github.com/griesemer/tryhard)：try 设计草案的实验性实现

不过一些知名 Go 程序员却给出了和调查结果**不一样的观点**。比如：Go 社区的知名 Go 程序员 [Dave Cheney](https://dave.cheney.net/) 就在其博客文章[“Go 语言之禅”](https://dave.cheney.net/2020/02/23/the-zen-of-go)中直言不讳地阐述了自己的观点：

```go
Go的成功很大程度上要归功于显式的处理错误的方式，因为它让Go程序员首先考虑失败情况，这将引导Go程序员专注于在编写代码时处理故障，而不是在程序部署并运行在生产环境后再进行故障的处理。而反复出现的代码片段：

if err != nil {
	return err
}

所付出的成本已基本被在故障发生时刻去处理故障的成本超过了。 
```

Go 语言之父 Rob Pike 在 2019 年的[Go Sydney 聚会的演讲中谈及 Go2 的变化时](https://youtu.be/RIvL2ONhFBI)也认为`if err != nil`在代码库中的使用远没有少数人所建议的那么普遍。

著名 Go 培训师、[《Go 语言实战》](https://book.douban.com/subject/27015617/)联合作者之一的威廉·肯尼迪（William Kennedy）更是在 Go 团队[try 提案](https://github.com/golang/proposal/blob/master/design/32437-try-builtin.md)公示之后，发表了对[Go 社区的公开信](https://www.ardanlabs.com/blog/2019/07/an-open-letter-to-the-go-team-about-try.html)。他认为上面的 Go 用户调查结果(2018 年)可能夸大了人们对错误处理的抱怨，而这些变化可能并不是大多数 Go 开发人员真正想要或真正需要的。同时他希望 Go 团队不要接受[try 这个提案](https://www.ardanlabs.com/blog/2019/07/an-open-letter-to-the-go-team-about-try.html)，因为它引入了两种方法来完成相同的事情，这与 Go**完成一种事情仅有一种方法的原则**有背离，这样会导致代码库中出现严重的不一致（有些人使用新引入的 try，有些人则依旧喜欢使用传统的`if err != nil`的错误检查）。威廉姆甚至认为 Go 团队应该重新评估错误处理改善在 Go2 演进中的优先级，毕竟它在用户调查中仅排名第五位，并希望 Go 团队对调查数据进行重新评估。

以上两种观点交锋的结果是 Go 团队否决了大部分之前编写的关于 Go 错误处理改善的设计草案，仅“[Go2 错误检查(error inspecting)设计草案](https://github.com/golang/proposal/blob/master/design/go2draft-error-inspection.md)”中的部分内容最终在[Go 1.13 版本](https://tip.golang.org/doc/go1.13)中被接纳和实现。

## 2. 我们该怎么办？尽量去优化

神仙打架，作为吃瓜群众的我们究竟该怎么办呢？也许 Go 用户调查结果(2018 年)的确夸大了人们对错误处理的抱怨，但现实中大量反复出现的`if err != nil`的现象也的确是存在的。就连 Go 团队的技术负责人[Russ Cox](https://swtch.com/~rsc/)也承认**当前的 Go 错误处理机制对于 Go 开发人员来说的确会有一定的心智负担**。如果像上面例子（CopyFile）中那样编写错误处理代码，虽然功能正确，但显然错误处理不够简洁和优雅。

另外一名 Go 团队核心开发人员[Marcel van Lohuizen](https://github.com/mpvl)也对`if err != nil`的重复出现情况也进行了研究。他发现代码所在栈帧越低(越接近与 main 函数栈帧)，`if err != nil`就越不常见；反之，代码在栈中的位置越高（更接近于网络 I/O 操作或操作系统 API 调用），`if err != nil`出现的就越普遍，正如上面`CopyFile`那个例子中的情况：

![26 if err != nil 重复太多可以这么办](https://img-hello-world.oss-cn-beijing.aliyuncs.com/1c54c350d93054b76f32bf65bffb0dd0.png)

图7-2-2: if err != nil 反复出现在函数栈中的分布特点

不过该开发人员也认为：可以通过良好的设计减少或消除这类反复出现的错误检查。

好了！到这里我们可以确定一定以及肯定要对`if err != nil`的反复出现进行尽可能的优化，但是在未引入`try`或`check/handle`这些新语法的情况下，我们要怎么做呢？

## 3. 优化思路

优化反复出现的`if err != nil`代码块的本质目的是让错误检查和处理较少或不要干扰到正常业务代码，让正常业务代码看起来更具**视觉连续性**。我们大致有两个努力的方向：

- **改善代码的视觉呈现**：这个优化方法就好比给开发人员施加了某种“障眼法”，使得错误处理代码在开发者眼中的视觉呈现更为“优雅”。上面提到的 Go2 关于改善错误处理的几个技术草案本质上就是提供一种改善代码视觉呈现的语法糖。

比如：如果待优化的代码像下面这样：

```go
func SomeFunc() error {
	err := doStuff1()
	if err != nil {
		//handle error...
	}

	err = doStuff2()
	if err != nil {
		//handle error...
	}

	err = doStuff3()
	if err != nil {
		//handle error...
	}
} 
```

那么经由`try`技术草案优化后的代码将大致变成这样(由于 try 提案被否决，因此我们无法真实实现下面这样的错误处理)：

```go
func SomeFunc() error {
	defer func() {
		if err != nil {
			// handle error...
		}
	}()
	try(doStuff1())
	try(doStuff2())
	try(doStuff3())
} 
```

- **降低 if err != nil 重复的数量**：如果觉得`if err != nil`重复的次数过多，我们可以降低其出现次数，这其实是将该问题转换为降低函数/方法的复杂度了。

一个函数/方法内部出现多少个`if err != nil`才需要我们去优化和消除这种代码重复呢？这显然没有标准可言。我这里想到了一个粗略的评估方法：利用圈复杂度(Cyclomatic complexity)[http://kaelzhang81.github.io/2017/06/18/%E8%AF%A6%E8%A7%A3%E5%9C%88%E5%A4%8D%E6%9D%82%E5%BA%A6/]。圈复杂度是一种代码复杂度的衡量标准，我们常用它来衡量一个模块判定结构的复杂程度。圈复杂度大说明程序代码可能质量低且难于测试和维护，根据经验，程序的可能错误和高的圈复杂度有着很大关系。

圈复杂度可以通过程序控制流图计算，公式为：`V(G) = e + 2 - n`。其中：e 为控制流图中边的数量；n 为控制流图中节点的数量(包括起点和终点；所有终点只计算一次，多个 return 和 throw 算作一个节点)。下面是不同数量的 if 语句对应的不同圈复杂度：

![26 if err != nil 重复太多可以这么办](https://img-hello-world.oss-cn-beijing.aliyuncs.com/571caf4b8e800a927e9a90962cdc93de.png)

图7-2-3：if语句个数对函数圈复杂度的影响

我们看到三组 if 语句（不带 else）的圈复杂度已经达到了 4，两组 if 语句的圈复杂度为 3。因此这里给出一个建议值：对圈复杂度为 4 或 4 以上的模块代码进行重构优化，即如果一个函数/方法中的`if err != nil`数量为 3 个或 3 个以上，我们将尝试对其进行代码优化，以尝试减少或消除过多的`if err != nil`代码片段。当然这个建议值对于那些有代码洁癖的 gopher 来说可以全不作数_。

现实中的真实优化实施更多是上述两个方向的结合，这里用一幅四象限图来直观展示一下可能的优化思路：

![26 if err != nil 重复太多可以这么办](https://img-hello-world.oss-cn-beijing.aliyuncs.com/0c5943acc9c7f6ed85f7cab5bbd47e17.png)

图7-2-4：if err != nil优化思路的四象限图

下面我们分别来详细说明一下各个象限的优化思路(注：第三象限显然表示的是待优化的代码)。

### 1) 视觉扁平化

Go 提供了将触发错误处理的语句与错误处理代码放在一行的支持，比如上面的 SomeFunc 函数，我们可以将之等价重写为下面代码：

```go
func SomeFunc() error {
	if err := doStuff1(); err != nil { // handle error... }
	if err := doStuff2(); err != nil { // handle error... }
	if err := doStuff3(); err != nil { // handle error... }
} 
```

这虽然并未从本质上消除`if err != nil`代码块过多的问题，也没有降低 SomeFunc 的圈复杂度，但经过这种**视觉呈现上的优化**，多数 Gopher 会觉得代码看起来更舒服了。

不过这种优化显然是有约束的，如果错误处理分支的语句不是简单的`return err`，而是复杂如下面代码中这样：

```go
if _, err = io.Copy(w, r); err != nil {
	return fmt.Errorf("copy %s %s: %v", src, dst, err)
} 
```

那么"扁平化"会导致代码行过长，反倒降低了视觉呈现的“优雅度”。另外如果你使用`goimports`或`gofmt`工具对代码进行自动格式化，那么这些格式化工具会自动展开上述代码，这会让你困惑不已。

### 2) 重构：减少`if err != nil`的重复次数

我们沿着降低复杂度的方向对待优化代码进行重构，以减少`if err != nil`代码片段的重复次数。我们以上面的`CopyFile`为优化对象。 原 CopyFile 函数有 4 个重复出现的`if err != nil`代码段，这里我们将其减至 2 个。下面是一种优化方案的代码实现：

```go
// go-if-error-check-optimize-1.go

func openBoth(src, dst string) (*os.File, *os.File, error) {
	var r, w *os.File
	var err error
	if r, err = os.Open(src); err != nil {
		return nil, nil, fmt.Errorf("copy %s %s: %v", src, dst, err)
	}

	if w, err = os.Create(dst); err != nil {
		r.Close()
		return nil, nil, fmt.Errorf("copy %s %s: %v", src, dst, err)
	}
	return r, w, nil
}

func CopyFile(src, dst string) error {
	var err error
	var r, w *os.File
	if r, w, err = openBoth(src, dst); err != nil {
		return err
	}
	defer func() {
		r.Close()
		w.Close()
		if err != nil {
			os.Remove(dst)
		}
	}()

	if _, err = io.Copy(w, r); err != nil {
		return fmt.Errorf("copy %s %s: %v", src, dst, err)
	}
	return nil
} 
```

我们看到：为了减少 CopyFile 函数中的 if 检查的重复次数，我们引入一个中间层：`openBoth`函数。我们将打开源文件和创建目的文件的工作转移到了`openBoth`函数中。这样优化下来，CopyFile 的圈复杂度下降到我们可以接受的范围内，而新增的 openBoth 函数的圈复杂度也在可接受范围内。

### 3) check/handle 风格化

上面的位于第四象限的重构之法虽然减少了`if err != nil`代码片段的重复次数，但其视觉呈现依旧欠佳。Go2 的[check/handle 技术草案](https://github.com/golang/proposal/blob/master/design/go2draft-error-handling.md)的思路给了我们一些启发，我们可利用 panic 和 recover 封装一套跳转机制，模拟实现一套 check/handle 机制，在降低复杂度的同时，也能在视觉呈现上有所改善。我们仍然以`CopyFile`为例进行优化：

```go
// go-if-error-check-optimize-2.go
func check(err error) {
	if err != nil {
		panic(err)
	}
}

func CopyFile(src, dst string) (err error) {
	var r, w *os.File

	// error handler
	defer func() {
		if r != nil {
			r.Close()
		}
		if w != nil {
			w.Close()
		}
		if e := recover(); e != nil {
			if w != nil {
				os.Remove(dst)
			}
			err = fmt.Errorf("copy %s %s: %v", src, dst, err)
		}
	}()

	r, err = os.Open(src)
	check(err)

	w, err = os.Create(dst)
	check(err)

	_, err = io.Copy(w, r)
	check(err)

	return nil
} 
```

看一下这段 check/handle 风格的`CopyFile`代码，无论是从业务代码(Open -> Create -> Copy)的视觉连续性来看，还是从 CopyFile 的圈复杂度来看，这次优化显然都要好于前面的优化。这也再一次证实了现实中的真正好的优化更多是上述两个方向的结合。

不过这一优化方案也具有一定约束，比如：函数必须使用具名的 error 返回值、defer 性能(在 Go 1.14 版本中，与不使用 defer 的性能差异微乎其微，可忽略不计）、panic 和 recover 的性能等。尤其是 panic 和 recover 的性能要比正常函数返回的性能相差好多，下面是一个简单的性能基准对比测试：

```go
// panic_recover_performance_test.go 
package main

import (
	"errors"
	"testing"
)

func check(err error) {
	if err != nil {
		panic(err)
	}
}
func FooWithoutDefer() error {
	return errors.New("foo demo error")
}

func FooWithDefer() (err error) {
	defer func() {
		err = errors.New("foo demo error")
	}()
	return
}

func FooWithPanicAndRecover() (err error) {
	// error handler
	defer func() {
		if e := recover(); e != nil {
			err = errors.New("foowithpanic demo error")
		}
	}()

	check(FooWithoutDefer())
	return nil
}

func FooWithoutPanicAndRecover() error {
	return FooWithDefer()
}

func BenchmarkFuncWithoutPanicAndRecover(b *testing.B) {
	for i := 0; i < b.N; i++ {
		FooWithoutPanicAndRecover()
	}
}

func BenchmarkFuncWithPanicAndRecover(b *testing.B) {
	for i := 0; i < b.N; i++ {
		FooWithPanicAndRecover()
	}
} 
```

运行上述性能基准测试：

```shell
$ go test -bench . panic_recover_performance_test.go
goos: darwin
goarch: amd64
BenchmarkFuncWithoutPanicAndRecover-8   	39020437	        28.8 ns/op
BenchmarkFuncWithPanicAndRecover-8      	 4442336	       271 ns/op
PASS
ok  	command-line-arguments	2.639s 
```

我们看到 panic 和 recover 让函数调用的性能慢了近 10 倍。因此，我们在使用这种方案优化重复代码前，需要全面了解这些约束。

### 4) 封装：内置 error 状态

在 Go 语言之父 Rob Pike 的["Errors are values"](http://blog.golang.org/errors-are-values)一文中，Rob Pike 为我们呈现了 Go 标准库中使用了避免`if err != nil`反复出现的一种代码设计思路，bufio 包的 Writer 就是使用了这个思路实现的，因此它可以可以像下面这样使用：

```go
b := bufio.NewWriter(fd)
b.Write(p0[a:b])
b.Write(p1[c:d])
b.Write(p2[e:f])
// and so on
if b.Flush() != nil {
        return b.Flush()
    }
} 
```

我们看到上述代码中并没有判断三个 b.Write 的返回错误值，错误处理放在哪里了呢？我们打开一下$GOROOT/src/bufio/bufio.go，我们看到下面代码：

```go
// $GOROOT/src/bufio/bufio.go
type Writer struct {
    err error
    buf []byte
    n   int
    wr  io.Writer
}

func (b *Writer) Write(p []byte) (nn int, err error) {
    for len(p) > b.Available() && b.err == nil {
        ... ...
    }
    if b.err != nil {
        return nn, b.err
    }
    ......
    return nn, nil
} 
```

我们可以看到，错误状态被封装在 bufio.Writer 结构的内部了，Writer 定义了一个 err 字段作为一个内部错误状态值，它与 Writer 的实例绑定在了一起，并且在每次 Write 入口判断是否为 nil。一旦不为 nil，Write 其实什么都没做就返回了。

这种方法显然是消除`if err != nil`代码片段重复出现的理想方法。我们还以`CopyFile`为例，看看使用这种“内置 error 状态”的新封装方法后，我们能得到什么样的代码：

```go
// go-if-error-check-optimize-3.go
package main

import (
	"fmt"
	"io"
	"os"
)

type FileCopier struct {
	w   *os.File
	r   *os.File
	err error
}

func (f *FileCopier) open(path string) (*os.File, error) {
	if f.err != nil {
		return nil, f.err
	}

	h, err := os.Open(path)
	if err != nil {
		f.err = err
		return nil, err
	}
	return h, nil
}

func (f *FileCopier) openSrc(path string) {
	if f.err != nil {
		return
	}

	f.r, f.err = f.open(path)
	return
}

func (f *FileCopier) createDst(path string) {
	if f.err != nil {
		return
	}

	f.w, f.err = os.Create(path)
	return
}

func (f *FileCopier) copy() {
	if f.err != nil {
		return
	}

	if _, err := io.Copy(f.w, f.r); err != nil {
		f.err = err
	}
}

func (f *FileCopier) CopyFile(src, dst string) error {
	if f.err != nil {
		return f.err
	}

	defer func() {
		if f.r != nil {
			f.r.Close()
		}
		if f.w != nil {
			f.w.Close()
		}
		if f.err != nil {
			if f.w != nil {
				os.Remove(dst)
			}
		}
	}()

	f.openSrc(src)
	f.createDst(dst)
	f.copy()
	return f.err
}

func main() {
	var fc FileCopier
	err := fc.CopyFile("foo.txt", "bar.txt")
	if err != nil {
		fmt.Println("copy file error:", err)
		return
	}
	fmt.Println("copy file ok")
}
```

这次的重构很是彻底。我们将原 CopyFile 函数彻底抛弃，而重新将其逻辑封装到一个名为`FileCopier`结构的 CopyFile 方法中。FileCopier 结构内置了一个 err 字段用于保存内部的错误状态，这样在其 CopyFile 方法中，我们只需按照正常业务逻辑，顺序执行 openSrc、createDst 和 copy 即可，正常业务逻辑的视觉连续性就是这样被很好地实现的。同时该 CopyFile 方法的复杂度因 if 检查的“大量缺席”而变得很小。

## 4. 小结

本节要点：

- Go 使用显式错误结果和显式的错误检查是 Go 语言成功的重要因素，同时也是`if err != nil`反复出现的根本原因；
- 了解关于 Go 错误处理改善的两种观点；
- 了解减少和消除`if err != nil`代码片段的两个优化方向：改善视觉呈现与降低复杂度；
- 掌握错误处理代码优化的四种常见方法(位于三个不同象限中)，并根据所处场景与约束灵活使用。