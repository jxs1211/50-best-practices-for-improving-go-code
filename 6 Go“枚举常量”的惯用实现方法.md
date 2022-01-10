6 Go“枚举常量”的惯用实现方法

C 家族（C-Family）的主流编程语言，如 C++、Java 等都提供了定义枚举常量的语法。在 C 语言中，枚举是一个命名的整型常数的集合，下面是我们使用枚举定义的 Weekday 类型：

```c
// C语法
enum Weekday {
        SUNDAY,
        MONDAY,
        TUESDAY,
        WEDNESDAY,
        THURSDAY,
        FRIDAY,
        SATURDAY
};

int main() {
        enum Weekday d = SATURDAY;
        printf("%d\n", d); // 6
}
```

C 语言针对枚举类型还提供了很多语法上的便利，比如：如果没有显式给枚举常量赋初始值，那么枚举类型的第一个常量的值为 0，后续常量的值依次加 1。与使用 define 宏定义的常量相比，C 编译器可以对专用的枚举类型做严格的类型检查，使得程序更为安全。

枚举的存在代表了一类现实需求：

- 有限数量标识符构成的集合，且多数情况下并不关心集合中标识符实际对应的值
- 注重类型安全

与其他 C 家族主流语言（如 C++、Java）不同，Go 语言没有提供定义枚举常量的语法。我们通常使用常量语法定义枚举常量，比如用 Go 来定义上面的 Weekday：

```go
const (
        Sunday    = 0
        Monday    = 1
        Tuesday   = 2
        Wednesday = 3
        Thursday  = 4
        Friday    = 5
        Saturday  = 6
)
```

如果仅仅能支持到这种程度，那么 Go 就算不上是“站在巨人的肩膀上”了。首先 Go 的 const 语法提供了“隐式重复前一个非空表达式”的机制，比如下面代码：

```go
const (
        Apple, Banana = 11, 22
        Strawberry, Grape 
        Pear, Watermelon 
)
```

常量定义的后两行没有显式给予初始赋值，Go 编译器将为其隐式使用第一行的表达式，这样上述定义等价于：

```go
const (
        Apple, Banana = 11, 22
        Strawberry, Grape  = 11, 22
        Pear, Watermelon  = 11, 22
)
```

不过这显然无法满足“枚举”的要求。Go 在这个机制的基础上又提供了iota“神器”，有了 iota，我们就可以定义满足各种场景的枚举常量了。

iota 是 Go 语言的一个预定义标识符，它表示的含义是 const 声明块（包括单行声明）中每个常量所处位置在块中的偏移值（从零开始）。同时，每一行中的 iota 自身也是一个无类型常量，可以像上一节所提到的无类型常量那样自动参与到不同类型的求值过程中，而无需对其进行显式转型操作。

下面是 Go 标准库中 sync/mutex.go 中的一段枚举常量的定义：

```
// $GOROOT/src/sync/mutex.go (go 1.12.7)
const ( 
        mutexLocked = 1 << iota // mutex is locked
        mutexWoken
        mutexStarving
        mutexWaiterShift = iota
        starvationThresholdNs = 1e6
)
```

这是一个很典型的诠释 iota 含义的例子，我们逐行来看：

- mutexLocked = 1 << iota 这里是 const 声明块的第一行，iota 的值是该行在 const 块中的偏移，因此 iota 的值为 0，我们得到 mutexLocked 这个常量的值为 1 << 0，即 1；
- mutexWorken 这里是 const 声明块的第二行，由于没有显式的常量初始化表达式，根据 const 声明块的“隐式重复前一个非空表达式”的机制，该行等价于 mutexWorken = 1 << iota。该行为 const 块中的第二行，因此偏移量 iota 的值为 1，我们得到 mutexWorken 这个常量的值为 1<< 1，即 2；
- mutexStarving 该常量同 mutexWorken，该行等价于 mutexStarving = 1 << iota，由于在该行的 iota 的值为 2，因此我们得到 mutexStarving 这个常量的值为 1 << 2，即 4;
- mutexWaiterShift = iota 这一行的常量初始化表达式与前三行不同，由于该行为第四行，iota 的偏移值为 3，因此 mutexWaiterShift 的值就为 3。

位于同一行的 iota 即便出现多次，其值也是一样的：

```go
const (
        Apple, Banana = iota, iota + 10 // 0, 10 (iota = 0)
        Strawberry, Grape // 1, 11 (iota = 1)
        Pear, Watermelon  // 2, 12 (iota = 2)
)
```

如果我们要略过 iota = 0，而从 iota = 1 开始正式定义枚举常量，我们可以效仿下面代码：

```go
// $GOROOT/src/syscall/net_js.go go 1.12.7

const (
        _ = iota
        IPV6_V6ONLY
        SOMAXCONN
        SO_ERROR
)
```

如果我们要略过某一行，也可以使用类似方式:

```go
const (
        _ = iota // 0
        Pin1
        Pin2
        Pin3
        _
        Pin5    // 5   
)
```

iota 的加入让 Go 在枚举常量定义上面的表达力大增：

- 我们看到和传统 C 枚举常量相比，Go 提供的 iota 预定义标识符可以参与到常量的初始化表达式的计算中，这样我们可以以更为灵活的形式为枚举常量赋予初值。而传统 C 语言的枚举仅可以以已经定义了的常量参与到其他常量的初始值表达式中。比如：

```
//C代码：

enum Season {
        spring,
        summer = spring + 2,
        fall = spring + 3,
        winter = fall + 1
};
```

和带有 iota（行偏移值）的初始化表达式相比，在阅读上面这段代码时，如果要对 winter 进行求值，我们还要向上查询 fall 的值和 spring 的值。

- Go 的枚举常量可不限于整型值，定义浮点型枚举常量也可以。
  C 语言无法定义浮点类型的枚举常量，但 Go 语言可以，这其中的部分功劳要归功于 Go 无类型常量。

```go
const (
        PI   = 3.1415926              // π
        PI_2 = 3.1415926 / (2 * iota) // π/2
        PI_4                          // π/4   
)  
```

- iota 让你维护枚举常量“列表”更容易
  我们使用传统的枚举常量声明方式声明一组“颜色”常量：

```go
const ( 
        Black  = 1 
        Red    = 2
        Yellow = 3
)
```

常量按照首字母顺序排序。假如我们要增加一个颜色 Blue，根据字母序，这个新常量应该放在 Red 的前面，但这样一来，我们就需要将 Red 到 Yellow 的常量值都手动+1，十分费力。

```go
const (
        Blue   = 1
        Black  = 2
        Red    = 3
        Yellow = 4
)
```

我们使用 iota 重新定义这组“颜色”枚举常量：

```go
const (
        _ = iota     
        Blue
        Red 
        Yellow     
)
```

现在无论后期增加多少种颜色，我们只需将常量名插入到对应位置即可，其他无需做任何手工调整。

- 使用有类型枚举常量保证类型安全。
  枚举常量多数也是无类型常量。如果要严格考虑类型安全，可以定义有类型枚举常量。下面是 Go 标准库中一段定义有类型枚举常量的例子：

```go
// $GOROOT/src/time/time.go

type Weekday int

const (
        Sunday Weekday = iota
        Monday
        Tuesday
        Wednesday
        Thursday
        Friday
        Saturday
)
```

这样，后续要使用 Sunday~Saturday 这些有类型枚举常量，必须匹配 Weekday 类型的变量。

最后，我还要提一个“反例”。在一些枚举常量名称与其初始值有强烈对应关系的时候，枚举常量会直接使用显式数值作为常量的初始值，这样的情况极其少见，在 Go 标准库中我仅找到这一处：

```go
// $GOROOT/bytes/buffer.go

// Don't use iota for these, as the values need to correspond with the
// names and comments, which is easier to see when being explicit.
const (
        opRead      readOp = -1 // Any other read operation.
        opInvalid   readOp = 0  // Non-read operation.
        opReadRune1 readOp = 1  // Read rune of size 1.
        opReadRune2 readOp = 2  // Read rune of size 2.
        opReadRune3 readOp = 3  // Read rune of size 3.
        opReadRune4 readOp = 4  // Read rune of size 4.
)
```