7 go语言定义“零值可用”的类型

## 1. Go 类型的零值

作为 C 程序员出身的我，我总是喜欢用在使用 C 语言的”受过的苦“与 Go 语言中得到的”甜头“做比较，从而来证明 Go 语言设计者在当初设计 Go 语言时是做了充分考量的。

在 C99 规范中，有一段是否对栈上局部变量进行自动清零初始化的描述：

> 如果未显式初始化且具有自动存储持续时间的对象，则其值是不确定的。

规范的用语总是晦涩难懂的。这句话大致的意思就是：如果是在栈上分配的局部变量，且在声明时未对其进行显式初始化，那么这个变量的值是不确定的。比如：

```c
// varinit.c
#include <stdio.h>

static int cnt;

void f() {
    int n;
    printf("local n = %d\n", n);

    if (cnt > 5) {
        return;
    }

    cnt++;
    f();
}

int main() {
    f();
    return 0;
}
```

编译上面的程序并执行：

```c
// 环境 centos linux gcc 版本 4.1.2
// 注意：在您的环境中执行上述代码，输出的结果很大可能与这里有所不同
$ gcc varinit.c
$ ./a.out

local n = 0
local n = 10973
local n = 0
local n = 52
local n = 0
local n = 52
local n = 52
```

我们看到分配在栈上的未初始化变量的值是不确定的，虽然一些编译器的较新版本也都提供一些命令行参数选项用于对栈上变量进行零值初始化，比如 GCC 就提供如下命令行选项：

```c
-finit-local-zero
-finit-derived
-finit-integer=n
-finit-real=<zero|inf|-inf|nan|snan>
-finit-logical=<true|false>
-finit-character=n
```

但这并不能改变 C 语言原生不支持对未显式初始化局部变量进行零值初始化的事实。资深 C 程序员是深知这个陷阱带来的问题是有多严重的。因此同样出身于 C 语言的 Go 设计者们在 Go 中彻底对这个问题进行的修复和优化。根据[Go 语言规范](https://tip.golang.org/ref/spec#The_zero_value)：

> 当通过声明或调用new为变量分配存储空间，或者通过复合文字字面量或make调用创建新值，
> 并且还不提供显式初始化的情况下，Go会为变量或值提供默认值。

Go 语言的每种原生类型都有其默认值，这个默认值就是这个类型的零值。下面是 Go 规范定义的内置原生类型的默认值（零值）。

```
所有整型类型：0
浮点类型：0.0
布尔类型：false
字符串类型：""
指针、interface、slice、channel、map、function：nil
```

另外 Go 的零值初始是递归的，即诸如数组、结构体等类型的零值初始化就是对其组成元素逐一进行零值初始化。

## 2. 零值可用

我们现在知道了 Go 类型的零值，接下来我们来说“可用”。

Go 从诞生以来就秉承着尽量保持“零值可用”的理念，我们来看两个例子。

第一个例子是关于 slice 的：

```go
var zeroSlice []int
zeroSlice = append(zeroSlice, 1)
fmt.Println(zeroSlice) // 输出：[1]
```

我们声明了一个 []int 类型的 slice：zeroSlice，我们并没有对其进行显式初始化，这样 zeroSlice 这个变量被 Go 编译器置为零值：nil。按传统的思维，对于值为 nil 这样的变量我们要给其赋上合理的值后才能使用。但是 Go 具备零值可用的特性，我们可以直接对其使用 append 操作，并且不会出现引用 nil 的错误。

第二个例子是通过 nil 指针调用方法的：

```go
// callmethodthroughnilpointer.go
package main
  
import (
        "fmt"
        "net"
)

func main() {
        var p *net.TCPAddr
        fmt.Println(p) //输出：<nil>
}
```

我们声明了一个 net.TCPAddr 的指针变量，我们并未对其显式初始化，指针变量 p 会被 Go 编译器赋值为 nil。我们在标准输出上输出该变量，fmt.Println 会调用 p.String()。我们来看看 TCPAddr 这个类型的 String 方法实现：

```go
// $GOROOT/src/net/tcpsock.go
func (a *TCPAddr) String() string {
        if a == nil {
                return "<nil>"
        }
        ip := ipEmptyString(a.IP)
        if a.Zone != "" {
                return JoinHostPort(ip+"%"+a.Zone, itoa(a.Port))
        }
        return JoinHostPort(ip, itoa(a.Port))
}
```

我们看到 Go 标准库在定义 TCPAddr 类型以及其方法时充分考虑了“零值可用”的理念，使得通过值为 nil 的 TCPAddr 指针变量依然可以调用 String 方法。

在 Go 标准库和运行时代码中还有很多践行“零值可用”理念的好例子，最典型的莫过于 sync.Mutex 和 bytes.Buffer 了。

我们先来看看 sync.Mutex。在 C 语言中，如果我们要使用线程互斥锁，我们需要这么做：

```c
pthread_mutex_t mutex; // 不能直接使用

// 必须先进行初始化
pthread_mutex_init (&mutex, NULL);

// 然后才能执行lock或unlock
pthread_mutex_lock(&mutex); 
pthread_mutex_unlock(&mutex); 
```

但是在 Go 语言中，我们只需这么做：

```go
var mu sync.Mutex
mu.Lock()
mu.Unlock()
```

Go 标准库的设计者很“贴心”地将 sync.Mutex 结构体的零值状态设计为可用状态，这样让 Mutex 的调用者可以“省略”对 Mutex 的初始化而直接使用 Mutex。

Go 标准库中的 bytes.Buffer 亦是如此：

```go
// bytesbufferwrite.go
package main
  
import (
        "bytes"
)

func main() {
        var b bytes.Buffer
        b.Write([]byte("Effective Go"))
        fmt.Println(b.String()) // 输出：Effective Go
}
```

我们看到我们无需对 bytes.Buffer 类型的变量 b 进行任何显式初始化即可直接通过 b 调用其方法进行写入操作，这源于 bytes.Buffer 底层存储数据的是同样支持零值可用策略的 slice 类型：

```go
// $GOROOT/src/bytes/buffer.go
// A Buffer is a variable-sized buffer of bytes with Read and Write methods.
// The zero value for Buffer is an empty buffer ready to use.
type Buffer struct {
        buf      []byte // contents are the bytes buf[off : len(buf)]
        off      int    // read at &buf[off], write at &buf[len(buf)]
        lastRead readOp // last read operation, so that Unread* can work correctly.
}
```

## 3. 小结

Go 语言零值可用的理念给内置类型、标准库的使用者带来很多便利。不过 Go 并非所有类型都是零值可用的，并且零值可用也是有一定限制的，比如：slice 的零值可用不能通过下标形式操作数据：

```go
var s []int
s[0] = 12 // 报错！
s = append(s, 12) // OK
```

另外像 map 这样的内置类型也没有提供零值可用的支持：

```go
var m map[string]int
m["tonybai"] = 1 // 报错！

m1 := make(map[string]int)
m1["tonybai"] = 1 // OK
```

另外零值可用的类型要注意尽量避免值拷贝：

```go
var mu sync.Mutex
mu1 := mu // Error: 避免值拷贝
foo(mu) // Error: 避免值拷贝
```

我们可以通过指针方式传递类似 Mutex 这样的类型。

对于我们 Go 开发者而言，保持与 Go 一致的理念，给自定义的类型一个合理的零值，并坚持保持自定义类型是零值可用的，这样我们的 Go 代码会表现的更加符合 Go 惯用法。