12 go语言 init 函数的妙用

从程序逻辑结构角度来看，Go 包（package）是程序逻辑封装的基本单元，每个包都可以理解为一个”自治“的、封装良好的、对外部暴露有限接口的基本单元。一个 Go 程序就是由一组包组成的。

在 Go 包这一基本单元中分布着常量、包级变量、函数、类型和类型方法、接口等，我们要保证包内部的这些元素在被使用之前处于合理有效的初始状态，尤其是包级变量。在 Go 语言中，我们一般通过包的 init 函数来完成这一工作。

## 1. 认识 init 函数

Go 语言中有两个特殊的函数，一个是 main 包中的 main 函数，它是所有 Go 可执行程序的入口函数；另外一个就是包的 init 函数。

init 函数是一个无参数无返回值的函数：

```go
func init() {
	... ...
}
```

如果一个包定义了 init 函数，Go 运行时会负责在该包初始化时调用它的 init 函数。在 Go 程序中我们不能显式调用 init，否则会在编译期间报错：

```go
// call_init_in_main.go
package main

import "fmt"

func init() {
	fmt.Println("init invoked")
}

func main() {
	init()
}

$go run call_init_in_main.go 
# command-line-arguments
./call_init_in_main.go:10:2: undefined: init
```

一个 Go 包可以拥有多个 init 函数，每个组成 Go 包的 Go 源文件中亦可以定义多个 init 函数。在初始化该 Go 包时，Go 运行时会按照一定的次序逐一顺序地调用该包的 init 函数。Go 运行时不会并发调用 init 函数，它会等待一个 init 函数执行完毕返回后再执行下一个 init 函数，且每个 init 函数在整个 Go 程序生命周期内仅会被执行一次。因此，init 函数极其适合**做一些包级数据初始化工作以及包级数据初始状态的检查工作**。

一个包内的、分布在多个文件中的多个 init 函数的执行次序是什么样的呢？一般来说，先被传递给 Go 编译器的源文件中的 init 函数先被执行；同一个源文件中的多个 init 函数按声明顺序依次执行。但 Go 语言的惯例告诉我们：**不要依赖 init 函数的执行次序。**

## 2. 程序初始化顺序

init 函数为何适合做包级数据初始化工作以及包级数据初始状态的检查工作呢？除了 init 函数是顺序执行并仅被执行一次之外，Go 程序初始化顺序也给 init 函数提供了胜任该工作的前提条件。

Go 程序由一组包组合而成，程序的初始化就是这些包的初始化。每个 Go 包都会有自己的依赖包、每个包还包含有常量、变量、init 函数(其中 main 包有 main 函数)等，这些元素在程序初始化过程中的初始化顺序是什么样的呢？我们用下面的这幅图来说明一下：

![image-20220112204635851](C:\Users\xjshen\AppData\Roaming\Typora\typora-user-images\image-20220112204635851.png)

我们看到在上图中：

- main 包依赖 pkg1、pkg4 两个包；
- Go 运行时会根据包导入的顺序，先去初始化 main 包的第一个依赖包 pkg1；
- Go 运行时遵循**“深度优先”**原则查看到：pkg1 依赖 pkg2，于是 Go 运行时去初始化 pkg2；
- pkg2 依赖 pkg3，Go 运行时去初始化 pkg3；
- pkg3 没有依赖包，于是 Go 运行时在 pkg3 包中按照”常量 -> 变量 -> init 函数"的顺序进行初始化；
- pkg3 初始化完毕后，Go 运行时会回到 pkg2 并对 pkg2 进行初始化；接下来再回到 pkg1 并对 pkg1 进行初始化；
- 在调用完 pkg1 的 init 函数后，Go 运行时完成 main 包的第一个依赖包 pkg1 的初始化；
- Go 运行时接下来会初始化 main 包的第二个依赖包 pkg4；
- pkg4 的初始化过程与 pkg1 类似，也是先初始化其依赖包 pkg5，然后再初始化自身；
- 当 Go 运行时初始化完 pkg4 后，也就完成了对 main 包所有依赖包的初始化，接下来初始化 main 包自身；
- 在 main 包中，Go 运行时会按照”常量 -> 变量 -> init 函数"的顺序进行初始化，执行完这些初始化工作后才正式进入程序的入口函数 main 函数。

到这里，我们知道了 init 函数适合做包级数据初始化和初始状态检查的`前提条件就是 init 函数的执行顺位排在其所在包的包级变量之后`。

我们再通过代码示例来验证一下上述的程序启动初始化顺序：

```shell
// 示例程序的结构如下：

package-init-order
├── go.mod
├── main.go
├── pkg1
│   └── pkg1.go
├── pkg2
│   └── pkg2.go
└── pkg3
    └── pkg3.go
```

包的依赖关系如下：

- main 包依赖 pkg1 和 pkg3；
- pkg1 依赖 pkg2。

由于篇幅所限，这里仅列出 main 包的代码，pkg1、pkg2 和 pkg3 包的代码与 main 包类似：

```go
// package-init-order/main.go

package main

import (
	"fmt"

	_ "github.com/bigwhite/package-init-order/pkg1"
	_ "github.com/bigwhite/package-init-order/pkg3"
)

var (
	_  = constInitCheck()
	v1 = variableInit("v1")
	v2 = variableInit("v2")
)

const (
	c1 = "c1"
	c2 = "c2"
)

func constInitCheck() string {
	if c1 != "" {
		fmt.Println("main: const c1 init")
	}
	if c1 != "" {
		fmt.Println("main: const c2 init")
	}
	return ""
}

func variableInit(name string) string {
	fmt.Printf("main: var %s init\n", name)
	return name
}

func init() {
	fmt.Println("main: init")
}

func main() {
	// do nothing
}
```

我们看到 main 包并未使用 pkg1 和 pkg3 中的函数或方法，而是直接通过包的空别名方式“触发”pkg1 和 pkg3 的初始化，下面是这个程序的运行结果：

```shell
$go run main.go
pkg2: const c init
pkg2: var v init
pkg2: init
pkg1: const c init
pkg1: var v init
pkg1: init
pkg3: const c init
pkg3: var v init
pkg3: init
main: const c1 init
main: const c2 init
main: var v1 init
main: var v2 init
main: init
```

正如我们预期的那样，Go 运行时按照"pkg2 -> pkg1 -> pkg3 -> main"的包顺序以及在包内“常量” -> “变量” -> init 函数的顺序进行初始化。

## 3. 使用 init 函数检查包级变量的初始状态

init 函数就好比 Go 包真正投入使用之前的一个唯一的“质检员”，负责对包内部以及暴露到外部的包级数据（主要是包级变量）的初始状态进行检查。在 Go 运行时和标准库中，我们能发现很多 init 检查包级变量的初始状态的例子。

### a) 重置包级变量值

我们先看看标准库 flag 包的 init 函数：

```go
// $GOROOT/src/flag/flag.go

func init() {
        // Override generic FlagSet default Usage with call to global Usage.
        // Note: This is not CommandLine.Usage = Usage,
        // because we want any eventual call to use any updated value of Usage,
        // not the value it has when this line is run.
        CommandLine.Usage = commandLineUsage
}
```

CommandLine 是 flag 包的一个导出包级变量，它也是默认情况下(如果你没有新创建一个 FlagSet)代表命令行的变量，我们从其初始化表达式即可看出：

```go
var CommandLine = NewFlagSet(os.Args[0], ExitOnError)
```

CommandLine 的 Usage 字段在 NewFlagSet 函数中被初始化为 FlagSet 实例（也就是 CommandLine)的方法值(method value)：defaultUsage。如果一直保持这样，那么使用 Flag 默认 CommandLine 的外用用户就无法自定义 usage 输出了。于是 flag 包在 init 函数中，将 ComandLine 的 Usage 字段设置为一个包内未导出函数 commandLineUsage，后者则直接使用了 flag 包的另外一个导出包变量 Usage。这样通过 init 函数将 CommandLine 与包变量 Usage 关联在一起。当用户将自定义 usage 赋值给 Usage 后，就相当于改变了 CommandLine 变量的 Usage。

另外一个例子来自标准库的 context 包：

```go
// $GOROOT/src/context/context.go

// closedchan is a reusable closed channel.
var closedchan = make(chan struct{})

func init() {
        close(closedchan)
}
```

context 包在 cancelCtx 的 cancel 方法中需要一个可复用的、处于关闭状态的 channel，于是 context 包定义了一个未导出包级变量 closedchan 并对其进行了初始化。但初始化后的 closedchan 并不满足 context 包的要求，唯一能检查和更正其状态的地方就是 context 包的 init 函数，于是我们看到上面代码在 init 函数中将 closedchan 关闭了。

### b) 对包级变量进行初始化，保证其后续可用

有些包级变量需要一个较为复杂的初始化过程，简单的初始化表达式不能满足要求。而 init 函数则非常适合完成此项工作。

标准库 regexp 包的 init 函数就负责完成了对内部特殊字节数组的初始化，这个特殊字节数组被包内的 special 函数使用，用于判断某个字符是否需要转义：

```go
//  $GOROOT/src/regexp/regexp.go

// Bitmap used by func special to check whether a character needs to be escaped.
var specialBytes [16]byte

// special reports whether byte b needs to be escaped by QuoteMeta.
func special(b byte) bool {
	return b < utf8.RuneSelf && specialBytes[b%16]&(1<<(b/16)) != 0
}
  
func init() {
	for _, b := range []byte(`\.+*?()|[]{}^$`) {
        	specialBytes[b%16] |= 1 << (b / 16)
        }
}
```

标准库 net 包在 init 函数中对 rfc6724policyTable 这个未导出包级变量进行反转排序：

```go
// $GOROOT/src/net/addrselect.go
func init() {
        sort.Sort(sort.Reverse(byMaskLength(rfc6724policyTable)))
}
```

标准库 http 包则在 init 函数中根据环境变量 GODEBUG 的值对一些包级开关变量进行赋值：

```go
// $GOROOT/src/net/http/h2_bundle.go
var (
        http2VerboseLogs    bool
        http2logFrameWrites bool
        http2logFrameReads  bool
        http2inTests        bool
)

func init() {
        e := os.Getenv("GODEBUG")
        if strings.Contains(e, "http2debug=1") {
                http2VerboseLogs = true
        }
        if strings.Contains(e, "http2debug=2") {
                http2VerboseLogs = true
                http2logFrameWrites = true
                http2logFrameReads = true
        }
}
```

### c) init 函数中的“注册模式”

下面是使用lib/pq 包访问 PostgreSQL 数据库的一段代码示例：

```go
import (
	"database/sql"
	_ "github.com/lib/pq"
)

func main() {
	db, err := sql.Open("postgres", "user=pqgotest dbname=pqgotest sslmode=verify-full")
	if err != nil {
		log.Fatal(err)
	}

	age := 21
	rows, err := db.Query("SELECT name FROM users WHERE age = $1", age)
	...
}
```

对于初学 Go 的 gopher 来说，这是一段“神奇”的代码，因为在以空别名方式导入 lib/pq 包后，main 函数中似乎并没有使用 pq 的任何变量、函数或方法。这段代码的奥秘全在 pq 包的 init 函数中：

```go
// github.com/lib/pq/conn.go
... ...

func init() {
        sql.Register("postgres", &Driver{})
}
... ...
```

空别名方式导入 lib/pq 的副作用就是 Go 运行时会将 lib/pq 作为 main 包的依赖包，Go 运行时会初始化 pq 包，于是 pq 包的 init 函数得以执行。我们看到在 pq 包的 init 函数中，pq 包将自己实现的 sql 驱动(driver)注册到 sql 包中。这样只要应用层代码在 Open 数据库的时候传入驱动的名字（这里是“postgres”)，那么通过 sql.Open 函数返回的数据库实例句柄对数据库进行的操作实际上调用的都是 pq 这个驱动的相应实现。

这种通过在 init 函数中注册自己的实现的模式，降低了 Go 包对外的直接暴露，尤其是包级变量的暴露，避免了外部通过包级变量对包状态的改动。从 database/sql 的角度来看，这种“注册模式”实质是一种工厂设计模式的实现，sql.Open 函数就是该模式中的工厂方法，它根据外部传入的驱动名称“生产”出不同类别的数据库实例句柄。

这种“注册模式”在标准库的其他包中亦有广泛应用，比如：使用标准库 image 包获取各种格式的图片的宽和高：

```go
// get_image_size.go
package main

import (
        "fmt"
        "image"
        _ "image/gif"
        _ "image/jpeg"
        _ "image/png"
        "os"
)

func main() {
        // 支持png, jpeg, gif
        width, height, err := imageSize(os.Args[1])
        if err != nil {
                fmt.Println("get image size error:", err)
                return
        }
        fmt.Printf("image size: [%d, %d]\n", width, height)
}

func imageSize(imageFile string) (int, int, error) {
        f, _ := os.Open(imageFile)
        defer f.Close()

        img, _, err := image.Decode(f)
        if err != nil {
                return 0, 0, err
        }

        b := img.Bounds()
        return b.Max.X, b.Max.Y, nil
}
```

这个小程序支持 png、jpeg、gif 三种格式的图片，而达成这一目标正是因为 image/png、image/jpeg 和 image/gif 包在各自的 init 函数中将自己“注册”到 image 的支持格式列表中了：

```go
// $GOROOT/src/image/png/reader.go
func init() {
        image.RegisterFormat("png", pngHeader, Decode, DecodeConfig)
}

// $GOROOT/src/image/jpeg/reader.go
func init() {
        image.RegisterFormat("jpeg", "\xff\xd8", Decode, DecodeConfig)
}

// $GOROOT/src/image/gif/reader.go
func init() {
        image.RegisterFormat("gif", "GIF8?a", Decode, DecodeConfig)
}
```

### d) init 函数中检查失败的处理方法

init 函数是一个无参数无返回值的函数，并且它的主要目的就是保证其所在包在被正式使用之前包的初始状态是有效的。一旦 init 函数在检查包数据初始状态时遇到失败或错误的情况(尽管极少出现），则说明对包的“质检”亮了红灯，如果让包“出厂”，那么只会导致更为严重的影响。因此，在这种情况下，快速失败是最佳选择。我们一般建议直接调用 panic。或通过 log.Fatal 等方法记录异常日志后再调用 panic 使程序退出。

## 4. 小结

深入理解 init 函数，记住本节的几个要点即可：

- init 函数的几个特点：运行时调用、顺序、仅执行一次 。
- Go 程序的初始化顺序。
- init 函数是包出厂前的唯一“质检员”。