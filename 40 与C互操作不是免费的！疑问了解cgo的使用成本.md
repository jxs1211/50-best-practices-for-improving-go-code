40 与C互操作不是免费



## 与C互操作不是免费的！疑问了解cgo的使用成本

Go 语言有很强的 C 语言背景，除了语法具有继承性外，其设计者以及其设计目标都与 C 语言有着千丝万缕的联系。在 Go 语言与 C 语言的互操作 (Interoperability) 方面，Go 更是提供了强大的支持，尤其是在 Go 中使用 C 语言，你甚至可以直接在 Go 源文件中编写 C 代码，实现这种互操作的技术就是 **cgo**。

在如下一些场景中，我们很大可能甚至是不可避免地会使用到 cgo 来实现 Go 与 C 的互操作：

1. 为提升局部代码性能时，用 C 代码替换一些 Go 代码，C 代码之于 Go 就好比汇编代码之于 C；
2. 对 Go 内存 GC 的延迟敏感，需要自己手动进行内存管理 (分配和释放)；
3. 为一些专有的且没有 Go 替代品的 C 语言库制作 Go 绑定 (binding) 或包装。比如：Oracle 提供了 C 版本 `OCI` 库 (Oracle Call Interface)，但 Oracle 并未提供 Go 版本的 `OCI` 库以及连接数据库的协议细节，因此我们只能通过包装 C 语言的 OCI 版本的方式与 Oracle 数据库通信。类似的情况还包括一些图形驱动程序以及图形化的窗口系统接口 (比如：`OpenGL` 库等)；
4. 与遗留的且很难重写或替换的 C 代码进行交互。

使用 cgo 是需要付出一定成本的，且其复杂性也需要征服。我们先来回顾一下 cgo 的原理和使用方法。

## 1. Go 调用 C 代码的原理

下面是一个典型的采用了 cgo 的 Go 代码例子：

```go
// go-cgo/how_cgo_works.go 
package main

// #include <stdio.h>
// #include <stdlib.h>
//
// void print(char *str) {
//     printf("%s\n", str);
// }
import "C"

import "unsafe"

func main() {
	s := "Hello, Cgo"
	cs := C.CString(s)
	defer C.free(unsafe.Pointer(cs))
	C.print(cs) // Hello, Cgo
} 
```

与常规的 Go 代码相比，上述代码有几处 “特殊” 的地方：

- C 代码直接出现在 Go 源文件中，只是都以注释的形式存在；
- 紧邻注释了的 C 代码块之后 (**中间没有空行**)，我们导入了一个名为 `C` 的包；
- 在 main 函数中通过 `C` 这个包调用了 C 代码中定义的函数 `print`。

这就是在 Go 源码中调用 C 代码的语法。首先，Go 源码文件中的 C 代码是需要用注释包裹的，就像上面的 `include` 头文件以及 `print` 函数定义； 其次，`import "C"` 这行语句是必须的，而且其与上面的 C 代码之间不能用空行分隔，必须紧密相连。这里的 `“C”` 不是包名，而是一种类似名字空间的概念，也可以理解为**伪包名**，C 语言所有语法元素均在该伪包下面；最后，访问 C 语法元素时都要在其前面加上伪包 `C` 的前缀，比如 `C.uint` 和上面代码中的 `C.print`、`C.free` 等。

我们如何来编译这个带有 C 代码的 Go 源文件呢？其实与常规的 Go 源文件没啥区别，依旧可以直接通过 `go build` 或 `go run` 来编译和执行。我们通过 `go build -x -v` 输出带有 cgo 代码的 Go 源文件的构建细节：

```shell
$go build -x -v  how_cgo_works.go
WORK=/var/folders/cz/sbj5kg2d3m3c6j650z0qfm800000gn/T/go-build263412137
command-line-arguments
mkdir -p $WORK/b001/
cd sources/go-cgo
CGO_LDFLAGS='"-g" "-O2"' $GOROOT/pkg/tool/darwin_amd64/cgo -objdir $WORK/b001/ -importpath command-line-arguments -- -I $WORK/b001/ -g -O2 ./how_cgo_works.go
cd $WORK
clang -fno-caret-diagnostics -c -x c - -o /dev/null || true
... ...
cd $WORK/b001
TERM='dumb' clang -I sources/go-cgo -fPIC -m64 -pthread -fno-caret-diagnostics -Qunused-arguments -fmessage-length=0 -fdebug-prefix-map=$WORK/b001=/tmp/go-build -gno-record-gcc-switches -fno-common -I ./ -g -O2 -o ./_x001.o -c _cgo_export.c
TERM='dumb' clang -I go-cgo -fPIC -m64 -pthread -fno-caret-diagnostics -Qunused-arguments -fmessage-length=0 -fdebug-prefix-map=$WORK/b001=/tmp/go-build -gno-record-gcc-switches -fno-common -I ./ -g -O2 -o ./_x002.o -c how_cgo_works.cgo2.c
TERM='dumb' clang -I sources/go-cgo -fPIC -m64 -pthread -fno-caret-diagnostics -Qunused-arguments -fmessage-length=0 -fdebug-prefix-map=$WORK/b001=/tmp/go-build -gno-record-gcc-switches -fno-common -I ./ -g -O2 -o ./_cgo_main.o -c _cgo_main.c
cd sources/go-cgo
TERM='dumb' clang -I . -fPIC -m64 -pthread -fno-caret-diagnostics -Qunused-arguments -fmessage-length=0 -fdebug-prefix-map=$WORK/b001=/tmp/go-build -gno-record-gcc-switches -fno-common -o $WORK/b001/_cgo_.o $WORK/b001/_cgo_main.o $WORK/b001/_x001.o $WORK/b001/_x002.o -g -O2
TERM='dumb' $GOROOT/pkg/tool/darwin_amd64/cgo -dynpackage main -dynimport $WORK/b001/_cgo_.o -dynout $WORK/b001/_cgo_import.go
cat >$WORK/b001/_gomod_.go << 'EOF' # internal
package main
import _ "unsafe"
//go:linkname __debug_modinfo__ runtime.modinfo
... ...
	EOF
cat >$WORK/b001/importcfg << 'EOF' # internal
# import config
packagefile runtime/cgo=/Users/tonybai/Library/Caches/go-build/2a/2a260e9c1cb4edbd8fb5c85a17287e2634d09a354ca9637ae406bbc38e45eaa4-d
packagefile syscall=$GOROOT/pkg/darwin_amd64/syscall.a
packagefile runtime=$GOROOT/pkg/darwin_amd64/runtime.a
EOF
$GOROOT/pkg/tool/darwin_amd64/compile -o $WORK/b001/_pkg_.a -trimpath "$WORK/b001=>" -p main -lang=go1.12 -buildid XZhNopmEOOMBeqF-tlzW/XZhNopmEOOMBeqF-tlzW -goversion go1.14 -D _sources/go-cgo -importcfg $WORK/b001/importcfg -pack -c=4 $WORK/b001/_cgo_gotypes.go $WORK/b001/how_cgo_works.cgo1.go $WORK/b001/_cgo_import.go $WORK/b001/_gomod_.go
$GOROOT/pkg/tool/darwin_amd64/pack r $WORK/b001/_pkg_.a $WORK/b001/_x001.o $WORK/b001/_x002.o # internal
$GOROOT/pkg/tool/darwin_amd64/buildid -w $WORK/b001/_pkg_.a # internal
cp $WORK/b001/_pkg_.a /Users/tonybai/Library/Caches/go-build/9e/9e66ab5998e6462258e3e5d88f018c8be69c8b6cde48f3f302fbbf4c31378d05-d # internal
cat >$WORK/b001/importcfg.link << 'EOF' # internal
packagefile command-line-arguments=$WORK/b001/_pkg_.a
packagefile runtime/cgo=/Users/tonybai/Library/Caches/go-build/2a/2a260e9c1cb4edbd8fb5c85a17287e2634d09a354ca9637ae406bbc38e45eaa4-d
packagefile syscall=$GOROOT/pkg/darwin_amd64/syscall.a
packagefile runtime=$GOROOT/pkg/darwin_amd64/runtime.a
... ...
EOF
mkdir -p $WORK/b001/exe/
cd .
$GOROOT/pkg/tool/darwin_amd64/link -o $WORK/b001/exe/a.out -importcfg $WORK/b001/importcfg.link -buildmode=exe -buildid=jvyV1pNmS_VCzc6EKmZA/XZhNopmEOOMBeqF-tlzW/DozBr2yfhmAPOP7dHs1R/jvyV1pNmS_VCzc6EKmZA -extld=clang $WORK/b001/_pkg_.a
$GOROOT/pkg/tool/darwin_amd64/buildid -w $WORK/b001/exe/a.out # internal
mv $WORK/b001/exe/a.out how_cgo_works
rm -r $WORK/b001/ 
```

构建过程输出的内容很多，我们用一幅图来描绘一下总体脉络：

![40 与C互操作不是免费的！疑问了解cgo的使用成本](https://img-hello-world.oss-cn-beijing.aliyuncs.com/89bf98d2edef9b466592203292a85dae.png)

图 9-7-1: cgo 程序的构建过程

我们看到在实际编译过程中，`go build` 调用了名为 `cgo` 的工具，`cgo` 会识别和读取 Go 源文件 (`how_cgo_works.go`) 中的 C 代码，并将其提取后交给外部的 C 编译器 (`clang或gcc`) 编译，最后与 Go 源码编译后的目标文件链接成一个可执行程序。这样我们就不难理解为何 Go 源文件中的 C 代码要用注释包裹并放在 `C` 这个伪包下面了，这些特殊的语法都是可以被 `cgo` 识别并使用的。

## 2. 在 Go 中使用 C 语言的类型

### 1) 原生类型

- 数值类型

在 Go 中可以用如下方式访问 C 原生的数值类型：

```go
C.char,
C.schar (signed char),
C.uchar (unsigned char),
C.short,
C.ushort (unsigned short),
C.int, C.uint (unsigned int),
C.long,
C.ulong (unsigned long),
C.longlong (long long),
C.ulonglong (unsigned long long),
C.float,
C.double 
```

Go 的数值类型与 C 中的数值类型不是一一对应的。因此在使用对方类型变量时少不了显式转型操作，如 [Go 官方一篇博客文章](https://blog.golang.org/cgo)中的这个例子：

```go
// #include <stdlib.h>
import "C"

func Random() int {
    return int(C.random())//C.long -> Go的int
}

func Seed(i int) {
    C.srandom(C.uint(i))//Go的uint -> C的uint
}
... ... 
```



- 指针类型

原生数值类型的指针类型可按 Go 语法在类型前面加上星号 *，比如：`var p *C.int`。 但 `void*` 比较特殊，在 Go 中用 `unsafe.Pointer` 表示它，这是因为任何类型的指针值都可以转换为 `unsafe.Pointer` 类型，而 `unsafe.Pointer` 类型也可以转换回任意类型的指针类型。

- 字符串类型

C 语言中并不存在原生的字符串类型，在 C 中用带结尾`'\0'` 的字符数组来表示字符串；而在 Go 中，`string` 类型是语言的原生类型，因此这两种语言互操作是势必要做字符串类型的转换的。

通过 `C.CString` 函数，我们可以将 Go 的 string 类型转换为 C 的 “字符串” 类型后再传给 C 函数使用。就如我们在本文开篇例子中使用的那样：

```go
s := "Hello, Cgo\n"
cs := C.CString(s)
C.print(cs) 
```

不过这个转型相当于在 C 语言世界的堆上分配一块新内存空间，这样转型后所得到的 C 字符串 `cs` 并不能由 Go 的 GC (垃圾回收器) 所管理，我们必须在使用后手动释放 `cs` 所占用的内存，这就是为何例子中通过 `defer` 调用 `C.free` 释放掉 `cs` 的原因。再次强调：**在 C 内部分配的内存，Go 中的 GC 是无法感知到的，因此要记着使用后手动释放**。

通过 `C.GoString` 可将 C 的字符串 (`*C.char`) 转换为 Go 的 string 类型，例如：

```go
// #include <stdio.h>
// #include <stdlib.h>
// char *foo = "hellofoo";
import "C"

import "fmt"

func main() {
    ... ...
    fmt.Printf("%T\n", C.GoString(C.foo)) // string
} 
```

这相当于在 Go 世界重新分配一块内存对象，并复制了 C 的字符串 (`foo`) 的信息，后续这个位于 Go 世界的新的 string 类型对象将和其他 Go 对象一样接受 GC 的管理。

- 数组类型

C 语言中的数组与 Go 语言中的数组差异较大，后者是原生的值类型，而前者与 C 中的指针大部分场合都可以随意转换。Go 仅提供了 `C.GoBytes` 用于将 C 中的 `char` 类型数组转换为 Go 中的 `[]byte` 切片类型：

```go
// go-cgo/c_char_array_to_go_byte_slice.go 
package main

// char cArray[] = {'a', 'b', 'c', 'd', 'e', 'f', 'g'};
import "C"
import (
	"fmt"
	"unsafe"
)

func main() {
	goArray := C.GoBytes(unsafe.Pointer(&C.cArray[0]), 7)
	fmt.Printf("%c\n", goArray) // [a b c d e f g]
} 
```

而对于其他类型的 C 数组，目前似乎无法直接显式地在两者之间进行转型。我们可以通过特定转换函数来将 C 的特定类型数组转换为 Go 的切片类型 (Go 中数组是值类型，其大小是静态的，转换为切片更为通用一些)，下面是一个将 C 整型数组转换为 Go`[]int32` 切片类型的例子：

```go
// go-cgo/c_array_to_go_slice.go 
package main

// int cArray[] = {1, 2, 3, 4, 5, 6, 7};
import "C"
import (
	"fmt"
	"unsafe"
)

func CArrayToGoArray(cArray unsafe.Pointer, elemSize uintptr, len int) (goArray []int32) {
	for i := 0; i < len; i++ {
		j := *(*int32)((unsafe.Pointer)(uintptr(cArray) + uintptr(i)*elemSize))
		goArray = append(goArray, j)
	}
	return
}

func main() {
	goArray := CArrayToGoArray(unsafe.Pointer(&C.cArray[0]), unsafe.Sizeof(C.cArray[0]), 7)
	fmt.Println(goArray) // [1 2 3 4 5 6 7]
} 
```

这里要注意的是：Go 编译器并不能将 C 的 `cArray` 自动转换为数组的地址，所以不能像在 C 中使用数组那样将数组变量直接传递给函数，而是将数组第一个元素的地址传递给函数。

### 2) 自定义类型

除了原生类型外，我们还可以访问 C 代码中的自定义类型。

- 枚举 (enum)

```go
// go-cgo/c_enum.go 
package main

// enum color {
//    RED,
//    BLUE,
//    YELLOW
// };
import "C"
import "fmt"

func main() {
	var e, f, g C.enum_color = C.RED, C.BLUE, C.YELLOW
	fmt.Println(e, f, g) // 0 1 2
} 
```

对于具名的 C 枚举类型 `xx`，我们可以通过 `C.enum_xx` 来访问该类型。如果是匿名枚举，则只能访问其字段了。

- 结构体 (struct)

和枚举 enum 类似，我们可以通过 `C.struct_xx` 来访问 C 中定义的结构体类型 `xx`：

```go
// go-cgo/c_struct.go 
package main

// #include <stdlib.h>
//
// struct employee {
//     char *id;
//     int  age;
// };
import "C"

import (
	"fmt"
	"unsafe"
)

func main() {
	id := C.CString("1247")
	defer C.free(unsafe.Pointer(id))

	var p = C.struct_employee{
		id:  id,
		age: 21,
	}
	fmt.Printf("%#v\n", p) // main._Ctype_struct_employee{id:(*main._Ctype_char)(0x9800020), age:21, _:[4]uint8{0x0, 0x0, 0x0, 0x0}}
} 
```

- 联合体 (union)

下面我们尝试用与访问 C 的 `struct` 类型相同的方法来访问一个 C 的 `union` 类型：

```go
// go-cgo/c_union_1.go 
package main

// #include <stdio.h>
// union bar {
//        char   c;
//        int    i;
//        double d;
// };
import "C"
import "fmt"

func main() {
	var b *C.union_bar = new(C.union_bar)
	b.c = 4
	fmt.Println(b)
} 
```

但在编译时，Go 却报了如下错误：

```
$go run c_union_1.go 
# command-line-arguments
./c_union_1.go:14:3: b.c undefined (type *[8]byte has no field or method c) 
```

从报错的信息来看，Go 对待 C 的 `union` 类型与其他类型不同，Go 将 `union` 类型看成 `[N]byte` 来对待，其中 `N` 为 `union` 类型中最长字段的大小 (圆整后的)：

```go
// go-cgo/c_union_2.go 
func main() {
    var b *C.union_bar = new(C.union_bar)
    b[0] = 4 
    fmt.Println(b) // &[4 0 0 0 0 0 0 0]
} 
```



- 别名类型 (typedef)

在 Go 中访问 C 中使用 `typedef` 定义的别名类型时，其访问方式与原类型的访问方式相同：

```go
// typedef int myint;

var a C.myint = 5
fmt.Println(a)

// typedef struct employee myemployee;

var m C.struct_myemployee 
```

从例子中可以看出，对原生类型的别名，直接访问这个新类型名即可。而对于复合类型的别名，需要根据原复合类型的访问方式对新别名进行访问，比如 `myemployee` 的实际类型为 `struct`，那么使用 `myemployee` 时也要加上 `struct_`前缀。

### 3) Go 中获取 C 类型大小

为了方便获得 C 世界中的类型的大小，Go 提供了 `C.sizeof_T` 来获取 `C.T` 类型的大小。如果是结构体、枚举以及联合体类型，我们需要在 `T` 前面加上 `struct_`、`enum_`和 `union_`的前缀，就像下面例子中这样：

```go
// go-cgo/c_type_size.go 
package main

// struct employee {
//     char *id;
//     int  age;
// };
import "C"

import (
	"fmt"
)

func main() {
	fmt.Printf("%#v\n", C.sizeof_int)             // 4
	fmt.Printf("%#v\n", C.sizeof_char)            // 1
	fmt.Printf("%#v\n", C.sizeof_struct_employee) // 16
} 
```

## 3. 在 Go 中链接外部 C 库

在上面的例子中我们已经演示了在 Go 中是如何访问 C 的类型、变量和函数的，一般方法就是加上 `C` 前缀即可，对于 C 标准库中的函数尤其是这样。不过虽然我们可以在 Go 源码文件中直接定义 C 类型、变量和 C 函数，但从代码结构上来讲，在 Go 源文件中大量编写 C 代码可不是什么 Go 推荐的惯用法。那如何将 C 函数和变量定义从 Go 源码中分离出去单独定义呢？我们很容易想到将 C 的代码以共享库的形式提供给 Go 源码。

Go 提供了`#cgo` 指示符可以指定 Go 源码在编译后与哪些共享库进行链接。我们来看一下例子：

```go
// go-cgo/foo.go
package main

// #cgo CFLAGS: -I${SRCDIR}
// #cgo LDFLAGS: -L${SRCDIR} -lfoo
// #include <stdio.h>
// #include <stdlib.h>
// #include "foo.h"
import "C"
import "fmt"

func main() {
	fmt.Println(C.count)
	C.foo()
} 
```

我们看到上面例子中通过`#cgo` 指示符告诉 Go 编译器在当前源码目录 (`${SRCDIR}` 会在编译过程中自动转换为当前源码所在目录的绝对路径) 下查找头文件 `foo.h`，并链接当前源码目录下的 `libfoo` 共享库。`C.count` 变量和 `C.foo` 函数的定义都在 `libfoo` 共享库中。我们来创建这个共享库：

```c
// go-cgo/foo.h

extern int count;
void foo();

// go-cgo/foo.c
#include "foo.h"

int count = 6;
void foo() {
    printf("I am foo!\n");
}

$gcc -c foo.c
$ar rv libfoo.a foo.o 
```

我们用 `ar` 工具成功创建了一个静态共享库文件 `libfoo.a`，接下来我们构建并运行一下 `foo.go`：

```go
$go build foo.go
$./foo
6
I am foo! 
```

我们看到 `foo.go` 成功链接到 `libfoo.a` 并生成最终的二进制文件 `foo`。

Go 同样也支持链接动态共享库，我们用下面命令将上面的 `foo.c` 编译为一个动态共享库：

```go
$gcc -c foo.c
//$gcc -shared -Wl,-soname,libfoo.so -o libfoo.so  foo.o (在linux上)
$gcc -shared -o libfoo.so  foo.o 
```

我们重新编译 `foo.go`，并查看 (在 linux 上可以使用 `ldd`，在 macOS 上使用 `otool`) 重新生成的二进制文件 `foo` 的动态共享库依赖情况：

```shell
$> go build foo.go
$otool -L foo   
foo:
	libfoo.so (compatibility version 0.0.0, current version 0.0.0)
	/usr/lib/libSystem.B.dylib (compatibility version 1.0.0, current version 1252.250.1) 
```

有一点值得注意，那就是 Go 支持多返回值，而 C 并不支持，因此当将 C 函数用在多返回值的 Go 调用中时，C 的 `errno` 将作为函数返回值列表中最后那个 `error` 返回值返回，下面是个例子：

```go
// go-cgo/c_errno.go 
package main

// #include <stdlib.h>
// #include <stdio.h>
// #include <errno.h>
// int foo(int i) {
//    errno = 0;
//    if (i > 5) {
//        errno = 8;
//        return i - 5;
//    } else {
//        return i;
//    }
//}
import "C"
import "fmt"

func main() {
	i, err := C.foo(C.int(8))
	if err != nil {
		fmt.Println(err)
	} else {
		fmt.Println(i)
	}
} 
```

运行这个例子：

```
$go run c_errno.go
exec format error 
```

`exec format error` 就是 `errno` 为 8 时的错误描述信息，我们可以在 C 运行时库的 `errno.h` 中找到 `errno=8` 与这段描述信息的联系：

```
#define ENOEXEC      8  /* Exec format error */ 
```

## 4. C 中使用 Go 函数

与在 Go 中使用 C 源码相比，在 C 中使用 Go 函数的场合极少。在 Go 中，可以使用 `export + 函数名`来导出 Go 函数为 C 所使用。目前 Go 对于导出函数供 C 使用的功能还十分有限，两种语言的调用约定不同，类型无法一一对应以及 Go 中类似垃圾回收这样的高级功能让导出 Go 函数这一特性难于完美实现，导出的函数依旧无法完全脱离 Go 的环境，因此其实用性大打折扣。

## 5. 使用 cgo 的成本

通过上面的学习，我们了解到 cgo 让我们可以在 Go 代码中使用 C 中的类型、变量和函数，但 Go 的这种经由 cgo 与 C 代码互操作的行为不是免费的，在本小节我们就来看看使用 cgo 都要付出哪些成本。

### 1) 不能忽略的调用成本

通过上述了解，在 Go 代码中调用 C 函数看起来似乎很 “平滑”，但实际上这种调用要比调用 Go 函数多付出一个甚至是多个**数量级**的成本。我们使用 Go 自带的性能基准测试来 “定量” 看看这种成本：

```go
// go-cgo/cgo-perf/cgo_call.go
package main

//
// void foo() {
// }
//
import "C"

func CallCFunc() {
	C.foo()
}

func foo() {

}

func CallGoFunc() {
	foo()
}

// go-cgo/cgo-perf/cgo_call_test.go 
package main

import "testing"

func BenchmarkCGO(b *testing.B) {
	for i := 0; i < b.N; i++ {
		CallCFunc()
	}
}

func BenchmarkGo(b *testing.B) {
	for i := 0; i < b.N; i++ {
		CallGoFunc()
	}
} 
```

运行这个基准测试，注意务必通过 `-gcflags '-l'` 关闭内联优化，这样才能得到 “公平” 的测试结果：

```go
$go test -bench . -gcflags '-l' 
goos: darwin
goarch: amd64
pkg: go-cgo/cgo-perf
BenchmarkCGO-8   	21359138	        56.5 ns/op
BenchmarkGo-8    	509841430	         2.37 ns/op
PASS
ok  	go-cgo/cgo-perf	2.716s 
```

通过结果我们看到：在这个例子中，通过 cgo 调用 C 函数付出的成本是调用 Go 函数的将近 30 倍。因此如果非要使用 `cgo`，一个不错的方案是将代码尽量下推到 C 中以减少调用次数，降低平均调用成本。

> 注意：`import "C"` 不支持放在 xx_test.go 文件中。

### 2) 增加线程 “暴涨” 的可能性

Go 以轻量级 goroutine 应对大并发而闻名，goroutine 和内核线程之间通过多路复用方式对应，这样通常 Go 应用会启动很多 goroutine，但创建的线程数量是有限的，通过下面例子我们可以印证这一点：

```go
// go-cgo/go_sleep.go
package main

import (
	"sync"
	"time"
)

func goSleep() {
	time.Sleep(time.Second * 1000)
}

func main() {
	var wg sync.WaitGroup
	wg.Add(100)
	for i := 0; i < 100; i++ {
		go func() {
			wg.Done()
			goSleep()
		}()
	}
	// 保证所有goroutine都已经启动
	wg.Wait()
	select {}
} 
```

我们在主 goroutine 之外还创建 100 个 goroutine，每个 goroutine 都睡眠 1000 秒，编译运行这个程序后，我们查看一下该进程中当前存在的线程数量：

```shell
以下在Ubuntu 18.04下执行(Go 1.14)：

$go build go_sleep.go
$./go_sleep

// 另外一个命令窗口输入：
$ps -ef|grep go_sleep
root     15829 10033  0 10:15 pts/0    00:00:00 ./go_sleep

$cat /proc/15829/status|grep -i thread
Threads:	3 
```

我们看到虽然我们额外启动了 100 个 goroutine，但进程使用的线程数仅为 3。这是因为 Go 优化了一些原本可以导致线程阻塞的 “系统调用”，就像上面的 `Sleep` 以及部分网络 I/O 操作，通过运行时调度实现了在不创建新线程的情况下依旧能达到同样的效果的目的。

但是 Go 调度器无法掌控 C 世界，如果将上面的 `Sleep` 换成 C 空间内的 `sleep` 函数调用，那么结果会是什么呢？我们来改编一下上面程序，让 goroutine 调用 C 空间的 `sleep` 函数：

```go
// go-cgo/cgo_sleep.go
package main

//#include <unistd.h>
//void cgoSleep() { sleep(1000); }
import "C"
import (
	"sync"
)

func cgoSleep() {
	C.cgoSleep()
}

func main() {
	var wg sync.WaitGroup
	wg.Add(100)
	for i := 0; i < 100; i++ {
		go func() {
			wg.Done()
			cgoSleep()
		}()
	}

	// 保证所有goroutine都已经启动
	wg.Wait()
	select {}
} 
```

我们编译运行这一改为调用 C 代码的示例程序：

```shell
$go build cgo_sleep.go
$./cgo_sleep

// 另外一个命令窗口输入：
$ps -ef|grep cgo_sleep
root     15939 10033  0 10:17 pts/0    00:00:00 ./cgo_sleep
$cat /proc/15939/status|grep -i thread
Threads:	103 
```

新创建的 goroutine 得到调度后，会执行 C 空间的 `sleep` 函数进入睡眠状态，这之后 Go 运行时调度代码无法抢占调用 C 代码的 goroutine 所在的线程，只能创建新的线程以供其他上没有绑定 `M` 的 `P` 上的 `goroutine` 使用（关于 `P`、`M` 的概念参考第 32 条 “了解 goroutine 的调度原理”），于是 100 个新线程被创建了出来。

虽然这是一种比较 “极端” 的情况，但日常开发中，我们很容易在 C 空间中写出导致线程阻塞的 C 代码，这会使得 Go 应用进程内线程数量暴涨的可能性大增，这与 Go 承诺的轻量级并发有背离。

### 3) 失去跨平台交叉构建能力 (cross compilation)

Go 有很多优点，比如：简单、原生支持并发等，而不错的可移植性也是 Go 被广大程序员接纳的重要因素之一。

在 [Go 1.7](http://tonybai.com/2016/06/21/some-changes-in-go-1-7/) 及以后版本中，我们可以通过下面命令查看 Go 支持操作系统和平台列表 (这里使用 Go 1.14)：

```shell
$go tool dist list
aix/ppc64
android/386
android/amd64
android/arm
android/arm64
... ...
js/wasm
linux/amd64
linux/arm
linux/arm64
... ...
linux/s390x
... ...
windows/386
windows/amd64
windows/arm 
```

随着 Go 支持的操作系统和平台的日益增多，Go 还为 Gopher 们提供了主流编程语言中最好的跨平台交叉编译能力，尤其是在 Go 1.5 版本及以后，使用 Go 进行跨平台交叉编译是极其简单的，我们仅需指定目标平台的操作系统类型 (`GOOS`) 和处理器架构类型 (`GOARCH`) 即可，就像下面例子这样：

```go
// 在macOS/amd64上
$GOOS=linux GOARCH=amd64 go build go_sleep.go 
```

但这种跨平台编译能力仅限于纯 Go 代码。如果我们跨平台编译使用了 cgo 技术的 Go 源文件，我们将得到如下类似的结果：

```
$GOOS=linux GOARCH=amd64 go build cgo_sleep.go
go: no Go source files 
```

当 Go 编译器执行跨平台编译时，它会将 `CGO_ENABLED` 置为 0，即关闭 cgo，我们显式开启 cgo 并再来跨平台编译一下上面的 `cgo_sleep.go` 文件：

```shell
$CGO_ENABLED=1 GOOS=linux GOARCH=amd64 go build ./cgo_sleep.go
# command-line-arguments
$GOROOT/pkg/tool/darwin_amd64/link: running clang failed: exit status 1
ld: warning: ignoring file /var/folders/cz/sbj5kg2d3m3c6j650z0qfm800000gn/T/go-link-231346986/go.o, file was built for unsupported file format ( 0x7F 0x45 0x4C 0x46 0x02 0x01 0x01 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 ) which is not the architecture being linked (x86_64): /var/folders/cz/sbj5kg2d3m3c6j650z0qfm800000gn/T/go-link-231346986/go.o
Undefined symbols for architecture x86_64:
  "_main", referenced from:
     implicit entry/start for main executable
ld: symbol(s) not found for architecture x86_64
clang: error: linker command failed with exit code 1 (use -v to see invocation) 
```

显然，即便显式开启 cgo，cgo 调用的 macOS 上的外部链接器 `clang` 因无法识别目标平台的目标文件格式而报错。

### 4) 其他成本

前面提到过，**Go 代码与 C 代码分别位于两个世界**，中间树立了高大的屏障，任一方都无法轻易跨过，这让它们无法很好利用对方的优势。

首当其冲的就是内存管理：Go 世界采用垃圾回收机制，而 C 世界采用手工内存管理，开发人员在 GC 与 “记着释放内存” 的规则间切换，极易产生 bug，给开发人员带来很大心智负担。

Go 所拥有的强大的工具链在 C 代码面前也是 “有劲儿使不上”，纵使 Go 的竞态检测工具、性能剖析工具、测试覆盖率工具、模糊测试以及源码竞态分析工具再怎么强大，也无法跨越 Go 与 C 之间的这个屏障。

由于 Go 工具无法轻易访问 C 世界代码，这也使得代码调试更加困难。辅助调试的运行时信息、行号、堆栈跟踪等信息一旦跨越屏障便消失地无影无踪。

## 6. 使用 cgo 代码的静态构建

所谓**静态构建**就是指构建后的应用其运行所需的所有符号、指令和数据都包含在自身的二进制文件当中，没有任何对外部动态共享库的依赖。静态构建出的二进制文件由于包含所有符号、指令和数据，因此其大小通常要比非静态构建的应用要大出许多。默认情况下，Go 没有采用静态构建。我们来看一个 Go 实现的文件服务器在默认情况下构建的例子：

```go
// go-cgo/server.go
package main

import (
    "log"
    "net/http"
    "os"
)

func main() {
    cwd, err := os.Getwd()
    if err != nil {
        log.Fatal(err)
    }

    srv := &http.Server{
        Addr:    ":8000", 
        Handler: http.FileServer(http.Dir(cwd)),
    }
    log.Fatal(srv.ListenAndServe())
} 
```

在 Linux 下 (ubuntu 18.04, Go 1.14) 使用默认条件构建该文件服务器：

```go
$go build -o server_default server.go
$ldd server_default
	linux-vdso.so.1 (0x00007ffde17ef000)
	libpthread.so.0 => /lib/x86_64-linux-gnu/libpthread.so.0 (0x00007f721b0f4000)
	libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f721ad03000)
	/lib64/ld-linux-x86-64.so.2 (0x00007f721b313000) 
```

我们看到默认构建出的 Go 应用有多个对外部动态共享库的依赖，这些依赖是怎么产生的呢？Go 应用由用户 Go 代码和 Go 标准库 / 运行时库组成，在这个程序中我们只能从标准库着手查找产生对外依赖的源头。

默认情况下，Go 的运行时环境变量 `CGO_ENABLED=1`(通过 `go env` 命令可以查看)，即默认开启 `cgo`，允许你在 Go 代码中调用 C 代码。Go 的预编译的标准库的`.a` 文件也是在这种情况下编译出来的。在 `$GOROOT/pkg/linux_amd64` 中，我们遍历所有预编译好的标准库`.a` 文件，并用 `nm` 输出每个`.a` 文件中的未定义符号 (状态为 `U`)，我们看到下面一些包是对外部有依赖的（动态链接）：

```shell
=> net.a
                 U _cgo_topofstack
                 U __errno_location
                 U getnameinfo
                 U _GLOBAL_OFFSET_TABLE_
                 U _cgo_topofstack
                 U __errno_location
                 U freeaddrinfo
                 U gai_strerror
                 U getaddrinfo
                 U _GLOBAL_OFFSET_TABLE_

=> os/user.a
                 U _GLOBAL_OFFSET_TABLE_
                 U malloc
                 U _cgo_topofstack
                 U free
                 U getgrgid_r
                 U getgrnam_r
                 U getpwnam_r
                 U getpwuid_r
                 U _GLOBAL_OFFSET_TABLE_
                 U realloc
                 U sysconf
                 U _cgo_topofstack
                 U getgrouplist
                 U _GLOBAL_OFFSET_TABLE_
... ... 
```

我们以 `os/user` 为例，在 `CGO_ENABLED=1`，即 cgo 开启的情况下，`os/user` 包中的 `lookupUserXXX` 系列函数采用了 Cgo 版本的实现，我们看到 `$GOROOT/src/os/user/cgo_lookup_unix.go` 源文件中的 `build tag` 中包含了 **+build cgo** 的构建指示器。这样在 `CGO_ENABLED=1` 的情况下即开启 `cgo` 的情况下该文件才会被编译，该文件中的 Cgo 版本实现的 `lookupUser` 将被使用：

```go
// $GOROOT/src/os/user/cgo_lookup_unix.go (go 1.14)

// +build aix darwin dragonfly freebsd !android,linux netbsd openbsd solaris
// +build cgo,!osusergo

package user
... ...
func lookupUser(username string) (*User, error) {
    var pwd C.struct_passwd
    var result *C.struct_passwd
    nameC := C.CString(username)
    defer C.free(unsafe.Pointer(nameC))
    ... ...
} 
```

这样一来凡是依赖上述包的 Go 代码最终编译的可执行文件都是要有外部依赖的，这就是默认情况编译出的 `server_default` 有外部依赖的原因 (`server_default` 至少依赖 `net.a`)。

这些 cgo 版本实现也都有对应的 Go 版本实现。还以 `user` 包为例，`user` 包的 `lookupUser` 函数的 Go 版本实现如下：

```go
// $GOROOT/src/os/user/lookup_unix.go (go 1.14)

// +build aix darwin dragonfly freebsd js,wasm !android,linux netbsd openbsd solaris
// +build !cgo osusergo

func lookupUser(username string) (*User, error) {
        f, err := os.Open(userFile)
        if err != nil {
                return nil, err
        }
        defer f.Close()
        return findUsername(username, f)
} 
```

那我们如何让编译器选择 Go 版本实现呢？从 `lookup_unix.go` 开头处的 `build tag` 中我们看到：通过设置 `CGO_ENABLED=0` 来关闭 `cgo` 是促使编译器选用 Go 版本实现的前提条件：

```shell
$CGO_ENABLED=0 go build -o server_static server.go
$ldd server_static
	not a dynamic executable
$nm server_static |grep " U " 
```

关闭 `cgo` 后，我们编译得到的 `server_static` 是一个静态编译的程序，它没有了对外部的任何依赖。

如果你使用 `go build` 的 `-x -v` 选项，你将看到 Go 编译器会重新编译依赖的包的静态版本，包括 `net` 等，并将编译后的`.a`(以包为单位) 放入编译器构建缓存目录下 (比如：`~/.cache/go-build/xxx`，后续可复用)，然后再静态连接这些版本。

那么问题来了：在 `CGO_ENABLED=1` 这个默认值的情况下，是否可以实现纯静态连接呢？答案是可以的。其原理也很简单，就是告诉链接器在最后的链接时采用静态链接方式，哪怕依赖的 Go 标准库中某些包使用的是 C 版本的实现。

[Go 官方文档介绍](https://www.helloworld.net/special/jnwqhy/9106661278)：Go 链接器支持两种工作模式 - `internal linking(内部链接)` 和 `external linking(外部链接)`。

所谓 `“internal linking”` 的大致意思是若用户代码中仅仅使用了 `net`、`os/user` 等几个标准库中的依赖 `cgo` 的包时，Go 链接器默认使用 `internal linking`，而无需启动外部链接器 (如：`gcc`、`clang` 等)。不过由于 Go 链接器功能有限，仅仅是将`.o` 和预编译好的标准库的`.a` 写到最终二进制文件中。因此如果标准库中是在 `CGO_ENABLED=1` 情况下编译的，那么编译出来的最终二进制文件依旧是动态链接的，即便在 `go build` 时传入 `-ldflags 'extldflags "-static"'` 亦无用，因为根本没有使用到外部链接器：

```shell
$go build -o server_internal_linking -ldflags '-extldflags "-static"' server.go
$ldd server_internal_linking 
	linux-vdso.so.1 (0x00007ffc58fab000)
	libpthread.so.0 => /lib/x86_64-linux-gnu/libpthread.so.0 (0x00007ff704544000)
	libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007ff704153000)
	/lib64/ld-linux-x86-64.so.2 (0x00007ff704763000) 
```

而 `external linking` 机制则是 Go 链接器将所有生成的`.o` 都打到一个`.o` 文件中，再将其交给外部的链接器，比如 `gcc` 或 `clang` 去做最终链接处理。如果此时，我们在 `go build` 的命令行参数中传入 `-ldflags 'extldflags "-static"'`，那么 `gcc/clang` 将会去做静态链接，将`.o` 中未定义 (undefined) 的符号都替换为真正的代码指令。我们可以通过 `-linkmode=external` 来强制 Go 链接器采用 `external linking`。还以 `server.go` 的编译为例：

```shell
$go build -o server-external-linking  -ldflags '-linkmode "external" -extldflags "-static"' server.go
$ldd server-external-linking 
	not a dynamic executable 
```

就这样，我们在 `CGO_ENABLED=1` 的情况下也编译构建出了一个纯静态链接的 Go 程序。如果你的用户层 Go 代码中使用了 Cgo 代码，那么 Go 链接器将会自动选择 `external linking` 的机制，我们以本节开始处的 `how_cgo_works.go` 为例：

```shell
$go build -o how_cgo_works_static  -ldflags '-extldflags "-static"' how_cgo_works.go
$ldd how_cgo_works_static 
	not a dynamic executable 
```

## 7. 小结

本节探讨了 cgo 的用途和原理、在 Go 中如何访问 C 语言的类型以及分析了 cgo 的使用成本，最后了解了如何在开启 cgo 的情况下实现 go 程序的静态构建。

牢记如下本节要点：

- 了解 cgo 的使用场景，尽量在不可避免的情况下才使用 cgo；
- 了解构建带有 cgo 程序的原理；
- 掌握 Go 中访问 C 元素的方法；
- cgo 是一柄双刃剑，一旦使用了 cgo，也要了解你要为此付出的成本；
- 了解 cgo 代码的构建要点：
  - 你的程序用了哪些标准库包？如果是 `net`、`os/user` 等几个依赖 cgo 实现的包之外的 Go 包，那么你的程序默认将是纯静态的，不依赖任何 C 运行时库等外部动态链接库；
  - 如果使用了 `net` 这样的包含 `cgo` 实现版本的标准库包，那么 `CGO_ENABLED` 的值将影响你的程序编译后的属性：是静态的还是动态链接的；
  - 在 `CGO_ENABLED=1` 的前提下且仅使用了 `net`、`os/user` 等依赖 cgo 实现的包，那么 `internal linking` 机制将被默认采用，编译过程不会采用静态链接；但如若依然要强制静态编译，需传递 `-ldflags '-linkmode "external" -extldflags "-static"'` 给 `go build` 命令。