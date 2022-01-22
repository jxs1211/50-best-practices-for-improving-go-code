37 time包，你用对了吗

## 42 time包，你用对了吗

时间作为人类认知世界的一把标尺，与人类活动密切相关。而软件已经渗透到人类生活的方方面面，伴随人类进行认知和演化。软件通过时间记录人类活动的发生时刻、维护人类活动秩序（比较时间的前后顺序）以及驱动着人类活动的良性运转。

我们现在编写的大部分现代应用程序都离不开**与时间相关的操作**。常见的时间操作包括：获取当前时间、时间比较、时区相关的时间操作、时间格式化、定时器（一次性定时器 timer 和重复定时器 ticker）的使用等。Go 语言通过标准库的 **time** 包为常见时间操作提供了全面的支持，在这一节中，我们就来一起解锁一下 Go 标准库 time 包使用的正确姿势。

## 1. 时间的基础操作

### 1) 获取当前时间

**获取当前时间**是最常用的时间操作。`time` 包提供了 `Now` 函数用于获取当前时间：

```go
// go-time-operations/get_current_time.go
package main
  
import (
        "fmt"
        "time"
)

func main() {
        t := time.Now()
        fmt.Println(t) //输出当前时间
} 
```

运行上述例子：

```go
$go run get_current_time.go 
2020-06-18 10:59:33.166871 +0800 CST m=+0.000073341 
```

我们看到：该 `Now` 函数以一个 `Time` 类型的结构体类型作为返回值。time 包将 `Time` 类型用作对一个 ** 即时时间 (time instant)** 的抽象。Go 1.14 版本中，`time.Time` 的结构如下：

```go
// $GOROOT/src/time.go(go1.14)

type Time struct {
        wall uint64
        ext  int64
        loc *Location
} 
```

由三个字段组成的 `Time` 结构体要同时表示两种时间：**挂钟时间 (wall time) \**和\**单调时间 (monotonic time) \**并且精度级别为\**纳秒 (nanosecond)**。

`Time` 结构体表示的这个抽象的 “挂钟时间” 主要用于**告知当前时间**。和我们日常真实使用的墙上的挂钟行为非常相似，它时快时慢，可以人为重新设定时间，比如：根据夏令时和冬令时对其进行调整或为了消除时钟误差、闰秒影响等对其进行调整。这和手动设定计算机时间或通过 NTP (网络时间协议) 同步调整挂钟时间十分相似。从其行为特征来看，连续两次通过 `Now` 函数获取的挂钟时间之间的差值不一定都是正值。在 [Go 1.9 版本](https://tonybai.com/2017/07/14/some-changes-in-go-1-9/)加入对[单调时间 (monotonic time)](https://tip.golang.org/doc/go1.9#monotonic-time) 的支持之前，[Cloudflare 公司](https://www.cloudflare.com/)的 DNS 系统就曾因两次采集的挂钟时间之差为负值 (遇到闰秒) 而出现过[严重故障](https://blog.cloudflare.com/how-and-why-the-leap-second-affected-cloudflare-dns/)。

而单调时间则是永远不会出现 “时间倒流” 现象的。单调时间表示的是程序进程启动之后**流逝的时间** ，两次采集的单调时间之差永远不可能为负数。Go 于 [1.9 版本](https://tonybai.com/2017/07/14/some-changes-in-go-1-9/)加入了对单调时间的支持，它常被用于**两个即时时间之间的比较和间隔计算**。

`time.Time` 结构体字段 `wall` 的最高比特位是一个名为 `hasMonotonic` 的**标志比特位**。当 `hasMonotonic` 被设置为 1 时，`time.Time` 表示的即时时间中既包含挂钟时间，也包含单调时间。下面是当 `Time` 同时包含这两种时间表示时 (`hasMonotonic` 比特位置为 1) 的原理示意图 (基于 Go 1.14 版本)：
![37 time包，你用对了吗](https://img-hello-world.oss-cn-beijing.aliyuncs.com/168d1034139c1dd0afc738f1ee3d1762.png)

图 9-4-1：当 hasMonotonic 为 1 时，time.Time 表示时间的原理

- `time.Time` 结构体中的 `wall` 字段表示**挂钟时间**，它是一个 64 位无符号整型。它的内部又被分成三段：分别表示 `hasMonotonic(1比特)`、秒数 (33 比特，挂钟时间的整秒数，距 1885 年 1 月 1 日的秒数) 和纳秒数 (30 比特，挂钟时间的非整秒数)；
- 而在 `hasMonotonic` 比特位为 1 的情况下，`ext` 字段则表示程序进程启动后的单调流逝时间，以纳秒为单位。
- loc 字段则是一个指向时区信息的指针。通过 `Now` 函数获取的即时时间是时区相关的。如果未显式指定时区，则默认使用系统时区。在 Linux/MacOS 上，默认使用的是 `/etc/localtime` 指向的时区数据：

```shell
$ls -l /etc/localtime
lrwxr-xr-x  1 root  wheel  39  7 25  2019 /etc/localtime@ -> /var/db/timezone/zoneinfo/Asia/Shanghai 
```

而当 `hasMonotonic` 为 0 时，`time.Time` 结构体仅表示挂钟时间，其原理如下图：

![37 time包，你用对了吗](https://img-hello-world.oss-cn-beijing.aliyuncs.com/b4c20514e09ea5a801874f18c44f5a2b.png)

图 9-4-2：当 hasMonotonic 为 0 时，time.Time 表示时间的原理

- `time.Time` 结构体中的 `wall` 字段的 `hasMonotonic(1比特)` 和秒数 (33 比特) 两部分均被置为 0，纳秒数 (30 比特) 依旧用于表示挂钟时间的非整数秒部分；
- 而在 `hasMonotonic` 比特位为 0 的情况下，`ext` 字段整个用于表示挂钟时间的整秒部分，其含义为距公元元年 1 月 1 日的秒数。
- loc 字段含义不变，依然是指向时区信息的指针。

通过 `time.Parse`、`time.Date` 或 `time.Unix` 构建的 `time.Time` 结构体，其中的 `hasMonotonic` 均为 0，即这样构建的 `Time` 实例仅表示挂钟时间，而没有单调时间。我们通过下面示例验证一下这点：

```go
// go-time-operations/construct_time_with_func_date.go 
package main

import (
	"fmt"
	"time"
	"unsafe"
)

func dumpWallAndExt(t time.Time) {
	var hasMonotonic int

	// 输出wall字段的值
	pWall := (*uint64)(unsafe.Pointer(&t))
	fmt.Printf("0x%X\n", *pWall)
	if (1<<63)&(*pWall) != 0 {
		hasMonotonic = 1
	}
	fmt.Printf("hasMonotonic = %d\n", hasMonotonic)

	// 输出ext字段的值
	pExt := (*int64)(unsafe.Pointer((uintptr(unsafe.Pointer(&t)) + unsafe.Sizeof(uint64(0)))))
	fmt.Printf("0x%X\n", *pExt)
	fmt.Printf("%d\n", *pExt/86400/365) // 粗略计算距今的年数
}

func constructTimeByDate() {
	loc, err := time.LoadLocation("Asia/Shanghai")
	if err != nil {
		fmt.Println("load time location failed:", err)
		return
	}
	t := time.Date(2020, 6, 18, 06, 0, 0, 10000, loc)
	fmt.Println(t)

	dumpWallAndExt(t)
}

func constructTimeByParse() {
	t, _ := time.Parse(time.RFC3339, "2020-06-18T06:00:00.00001+08:00")
	fmt.Println(t)
	dumpWallAndExt(t)
}

func main() {
	constructTimeByDate()
	constructTimeByParse()
} 
```

运行该示例：

```go
$go run construct_time_with_func_date.go 
2020-06-18 06:00:00.00001 +0800 CST
0x2710
hasMonotonic = 0
0xED67C8960
2020
2020-06-18 06:00:00.00001 +0800 CST
0x2710
hasMonotonic = 0
0xED67C8960
2020 
```

而通过 `time.Now` 函数获取的当前时间中则既包含挂钟时间，也包含单调时间：

```go
// go-time-operations/construct_time_with_func_now.go
... ...
func main() {
	t := time.Now()
	dumpWallAndExt(t)
}

$go run construct_time_with_func_now.go
0xBFB31A7A72BC57A0
hasMonotonic = 1
0x1164E
0 
```

`time.Now` 函数调用 `now` 函数获取系统即时时间，但在 `time` 包中 `now` 函数仅有一个原型声明，并没有函数体：

```go
// $GOROOT/src/time/time.go
func now() (sec int64, nsec int32, mono int64) 
```

`now` 函数的真正实现是 `runtime` 包的 `time_now` 函数，Go 链接器会将 `time_now` 链接为 `time.now`：

```go
// $GOROOT/src/runtime/timestub.go
... ...
//go:linkname time_now time.now
func time_now() (sec int64, nsec int32, mono int64) {
        sec, nsec = walltime()
        return sec, nsec, nanotime()
} 
```

`walltime` 和 `nanotime` 函数也都是 “过渡” 函数：

```go
// $GOROOT/src/runtime/time_nofake.go

//go:nosplit
func nanotime() int64 {
        return nanotime1()
}

func walltime() (sec int64, nsec int32) {
        return walltime1()
} 
```

真正获取系统时间的操作是在下面的汇编代码中通过系统调用 (system call) 实现的 (以 linux 为例):

```go
// $GOROOT/src/runtime/sys_linux_amd64.s
TEXT runtime·walltime1(SB),NOSPLIT,$8-12
... ...

noswitch:
        SUBQ    $16, SP         // Space for results
        ANDQ    $~15, SP        // Align for C code

        MOVQ    runtime·vdsoClockgettimeSym(SB), AX
        CMPQ    AX, $0
        JEQ     fallback
        MOVL    $0, DI // CLOCK_REALTIME
        LEAQ    0(SP), SI
        CALL    AX
... ...

TEXT runtime·nanotime1(SB),NOSPLIT,$8-8
... ...
noswitch:
        SUBQ    $16, SP         // Space for results
        ANDQ    $~15, SP        // Align for C code

        MOVQ    runtime·vdsoClockgettimeSym(SB), AX
        CMPQ    AX, $0
        JEQ     fallback
... ... 
```

### 2) 获取特定时区的当前时间

如果我们获取特定时区（而不是本地时区）的当前时间，我们可以使用下面几种方法。

- 设置 `TZ` 环境变量

`time.Now` 函数在获取当前时间时会考虑时区信息，如果 `TZ` 环境变量不为空，那么它将尝试读取该环境变量指定的时区信息并输出对应时区的即时时间表示。看下面示例，我们输出美国东部纽约所在时区的当前时间 (并对比北京时间)：

```go
$TZ=America/New_York go run get_current_time.go // 美国东部纽约时间
2020-06-18 16:13:55.703867 -0400 EDT m=+0.000064934

$go run get_current_time.go  //北京时间
2020-06-19 04:13:55.182577 +0800 CST m=+0.000068535 
```

如果 `TZ` 环境变量提供的时区信息有误或显式设置为 “”，`time.Now` 根据其值在时区数据库中找不到对应的时区信息，那么它将使用 UTC 时间 (Coordinated Universal Time, 国际协调时间)：

```go
$TZ=America/New_York1 go run get_current_time.go
2020-06-18 20:13:55.805963 +0000 UTC m=+0.000074006

$TZ="" go run get_current_time.go
2020-06-18 20:13:55.805963 +0000 UTC m=+0.000074006 
```

- 显式加载时区信息

如果不想设置 `TZ` 环境变量，我们也可以在代码中利用 `time` 包提供的 `LoadLocation` 函数显式加载特定时区信息，并将本地当前时间转换为特定时区的即时时间：

```go
// go-time-operations/get_time_with_tz.go 
package main

import (
	"fmt"
	"time"
)

func main() {
	t := time.Now()
	fmt.Println(t) //北京时间

	loc, err := time.LoadLocation("America/New_York")
	if err != nil {
		fmt.Println("load time location failed:", err)
		return
	}

	t1 := t.In(loc) // 转换成美国东部纽约时间表示
	fmt.Println(t1)
} 
```



```go
$go run get_time_with_tz.go 
2020-06-19 04:21:51.973803 +0800 CST m=+0.000171913
2020-06-18 16:21:51.973803 -0400 EDT 
```

显然这种方法也可用于任意时区间的时间转换，比如下面是美国东西部时区时间的转换：

```go
// go-time-operations/convert_time_between_tz.go 
package main

import (
	"fmt"
	"time"
)

func main() {
	locSrc, err := time.LoadLocation("America/Los_Angeles")
	if err != nil {
		fmt.Println("load time location failed:", err)
		return
	}

	t := time.Date(2020, 6, 18, 06, 0, 0, 0, locSrc)
	fmt.Println(t) // 美国西部洛杉矶时间，即太平洋时间

	locTo, err := time.LoadLocation("America/New_York")
	if err != nil {
		fmt.Println("load time location failed:", err)
		return
	}

	t1 := t.In(locTo) // 转换成美国东部纽约时间表示
	fmt.Println(t1)
} 
```

运行该示例：

```go
$go run convert_time_between_tz.go 
2020-06-18 06:00:00 -0700 PDT
2020-06-18 09:00:00 -0400 EDT 
```

### 3) 时间的比较与运算

从上面 `time.Time` 类型表示即时时间的原理我们知道：如果我们直接用 **== **和** != **来比较两个 `Time` 类型示例，那么参与比较的不仅包括挂钟时间，单调时间和时区信息也会一并参与到比较中，这样就会出现**在不同时区表示地球上的同一时刻的两个 `Time` 实例是不相等的 **，这违背了人的一贯认知。见下面示例：

```go
// go-time-operations/compare_two_time_with_operator.go 
package main

import (
	"fmt"
	"time"
)

func main() {
	t := time.Now()
	fmt.Println(t) //北京时间

	loc, err := time.LoadLocation("America/New_York")
	if err != nil {
		fmt.Println("load time location failed:", err)
		return
	}

	t1 := t.In(loc) // 转换成美国东部纽约时间表示
	fmt.Println(t == t1)
} 
```

运行该示例：

```
$go run compare_two_time_with_operator.go 
2020-06-19 10:05:53.16005 +0800 CST m=+0.000075281
2020-06-18 22:05:53.16005 -0400 EDT
false 
```

因此直接用 **== **和** !=** 来作比较是不适宜的，这也是为何 `time.Time` 类型不应该被用作 map 类型的 key 值的原因。`time.Time` 提供了 `Equal` 方法专用于对两个 `Time` 实例的比较：

```go
// $GOROOT/src/time/time.go (go 1.14)
func (t Time) Equal(u Time) bool {
        if t.wall&u.wall&hasMonotonic != 0 {
                return t.ext == u.ext
        }
        return t.sec() == u.sec() && t.nsec() == u.nsec()
} 
```

我们看到 `Equal` 方法的比较逻辑是这样的：当两个 `Time` 实例均带有单调时间数据时 (即 `hasMonotonic` 都为 1)，那么直接比较两者的**单调时间是否相等**；否则，分别比较两个时间的**整秒部分 (sec) \**和\**非整秒部分 (nsec)**，如果两个部分都相等，那么两个时间相同，否则不同。我们将上面的例子改为用 `Equal` 方法比较：

```go
// go-time-operations/compare_two_time_with_equal.go 
... ...
func main() {
	t := time.Now()
	fmt.Println(t) //北京时间

	loc, err := time.LoadLocation("America/New_York")
	if err != nil {
		fmt.Println("load time location failed:", err)
		return
	}

	t1 := t.In(loc) // 转换成美国东部纽约时间表示
	fmt.Println(t1)
	fmt.Println(t.Equal(t1))
} 
```

运行上述例子：

```
$go run compare_two_time_with_equal.go 
2020-06-19 10:21:25.474152 +0800 CST m=+0.000063104
2020-06-18 22:21:25.474152 -0400 EDT
true 
```

这回得到的结果与我们对时间的认知是一致的了。

`Time` 类型还提供 `Before` 和 `After` 方法用于判断两个即时时间的先后关系，和 `Equal` 的实现逻辑类似，它们的实现逻辑也是分成两种情形的，即两个即时时间都包含单调时间信息和除此之外的其他情况：

```go
// $GOROOT/src/time/time.go (go 1.14)

func (t Time) After(u Time) bool {
        if t.wall&u.wall&hasMonotonic != 0 {
                return t.ext > u.ext
        }
        ts := t.sec()
        us := u.sec()
        return ts > us || ts == us && t.nsec() > u.nsec()
}

func (t Time) Before(u Time) bool {
        if t.wall&u.wall&hasMonotonic != 0 {
                return t.ext < u.ext
        }
        return t.sec() < u.sec() || t.sec() == u.sec() && t.nsec() < u.nsec()
} 
```

除了上述对两个 `Time` 实例的比较关系操作提供了支持之外，我们还可以利用 `time` 包对两个即时时间进行时间运算，其中最主要的运算就是由 `Sub` 方法提供的差值运算 (`Since` 和 `Until` 方法也均是基于 `Sub` 方法实现的)，我们看一个例子：

```go
// go-time-operations/diff_two_time_with_sub.go 
package main

import (
	"fmt"
	"time"
)

func subTwoTimeHasMonotonic() {
	t1 := time.Now()
	time.Sleep(time.Second * 5)
	t2 := time.Now()
	diff := t2.Sub(t1)
	fmt.Printf("[hasMonotonic = 1] t2 - t1 = %v\n", diff)

}

func subTwoTimeNoMonotonic() {
	t1 := time.Date(2020, 6, 18, 0, 0, 0, 0, time.UTC)
	t2 := time.Date(2020, 6, 18, 12, 0, 0, 0, time.UTC)
	diff := t2.Sub(t1)
	fmt.Printf("[hasMonotonic = 0] t2 - t1 = %v\n", diff)
}

func main() {
	subTwoTimeHasMonotonic()
	subTwoTimeNoMonotonic()
} 
```

运行该示例：

```
$go run diff_two_time_with_sub.go 
[hasMonotonic = 1] t2 - t1 = 5.004840087s
[hasMonotonic = 0] t2 - t1 = 12h0m0s 
```

`Sub` 方法的返回值是 `time.Duration` 类型，这是一个纳秒值，上面的输出是其字符串化后的结果。和上面 `Equal` 的逻辑相似，`Sub` 方法对两个 `Time` 实例的差值处理也是分为两种情况：如果两个实例都含有单调时间信息 (`hasMonotonic=1`)，那么 `Sub` 方法直接返回两个实例的 `ext` 字段的差；否则，分别算出整秒部分的差与非整数秒部分的差，然后加和后返回。

## 2. 时间的格式化输出

时间的格式化输出是日常编程中经常遇到的 “题目”。以前在使用 [C 语言编程](https://tonybai.com/tag/c)时，我们使用 C 标准库提供的 `strftime` 对时间进行格式化，我们来回忆一下这样一段 C 代码：

```c
// go-time-operations/strftime_in_c.c
#include <stdio.h>
#include <time.h>

int main() {
        time_t now = time(NULL);

        struct tm *localTm;
        localTm = localtime(&now);

        char strTime[100];
        strftime(strTime, sizeof(strTime), "%Y-%m-%d %H:%M:%S", localTm);
        printf("%s\n", strTime);

        return 0;
} 
```

这段 c 代码输出结果是：

```
2029-06-18 16:07:00 
```



我们看到 `strftime` 采用 **“字符化”** 的占位符 (如：`%Y`、`%m等`) 拼接出时间的目标输出格式布局（如上面例子中的：`%Y-%m-%d %H:%M:%S`），这种方式不仅在 C 语言中被采用，很多其他主流编程语言也采用了该方案，比如：shell、[python](https://tonybai.com/tag/python)、[ruby](https://tonybai.com/tag/ruby)、[java](https://tonybai.com/tag/java) 等。这似乎已经成为了各种编程语言支持时间格式化输出的标准方案。这些占位符对应的字符（比如 Y、M、H）是对应英文单词的头母 (比如：Y 对应 Year 的头母)，因此相对来说也较容易记忆。

但是如果你在 Go 语言中使用类似 `strftime` 的这套 “标准”，看到输出结果的那一刻，你肯定要 “骂娘”:

```go
// go-time-operations/timeformat_in_c_way.go
package main

import (
    "fmt"
    "time"
)

func main() {
	fmt.Println(time.Now().Format("%Y-%m-%d %H:%M:%S"))
} 
```

上述 go 代码输出结果如下：

```
$go run timeformat_in_c_way.go 
%Y-%m-%d %H:%M:%S 
```

Go 居然将 “时间格式占位符字符串” 原封不动的输出了！

这是因为 Go 另辟了蹊径，采用了不同于 `strftime` 的时间格式化输出方案。Go 的设计者主要出于这样的考虑：虽然 `strftime` 的单个占位符使用了对应单词的首字母的形式，但是但真正写起代码来，不打开 `strftime` 函数的手册或查看[网页版的 strftime 助记符说明](http://strftime.org/)，很难真地拼出一个复杂的时间格式。并且对于一个 `"%Y-%m-%d %H:%M:%S"` 的格式串，如果不对照文档，很难在大脑中准确给出格式化后的时间结果。比如 `%Y` 和 `%y` 有何不同、`%M` 和 `%m` 又有何差别呢？

Go 语言采用了更为直观的 **“参考时间 (reference time)”** 替代 `strftime` 的各种标准占位符，使用 “参考时间” 构造出来的 “时间格式串” 与最终输出串是 “一模一样” 的，这就省去了程序员再次在大脑中对格式串进行解析的过程。我们通过下面例子看看 Go 方案的输出结果：

```go
// go-time-operations/timeformat_in_go_way.go 
package main

import (
	"fmt"
	"time"
)

func main() {
	fmt.Println(time.Now().Format("2006年01月02日 15时04分05秒"))
} 
```



运行该示例：

```
$go run timeformat_in_go_way.go 
2020年06月18日 12时27分32秒 
```

例子中我们使用的格式字符串：

```
"2006年01月02日 15时04分05秒" 
```

输出结果：

```
2020年06月18日 12时27分32秒 
```

是不是有点 “所见即所得” 的意味。

Go 文档中给出的标准的参考时间如下：

```
2006-01-02 15:04:05 PM -07:00 Jan Mon MST 
```

这个绝对时间本身并没有什么实际意义，仅是出于 **“好记”** 的考虑，我们将这个参考时间换为另外一种时间输出格式：

```
01/02 03:04:05PM '06 -0700 
```

我们看出 Go 设计者的 “用心良苦”！这个时间格式串恰好是**将助记符从小到大排序 (从 01 到 07) 的结果**，可以理解为： **01** 对应的是 `%M`, **02** 对应的是 `%d`，依次类推。下面这幅图形象地展示了 “参考时间”、“格式串” 与最终格式化的输出结果之间的关系：
![37 time包，你用对了吗](https://img-hello-world.oss-cn-beijing.aliyuncs.com/4f9c358c11010cbc8d7b4e0c00f1a0a4.png)

图 9-4-3：“参考时间”、“格式串” 与最终格式化的输出结果之间的关系

就笔者使用 Go 的经历来看，我在做时间格式化输出时，尤其是构建略微复杂的时间格式输出时，也还是要通过 `go doc time` 命令行或打开 `time` 包的[包帮助页面](https://tip.golang.org/pkg/time/)的。从社区的反馈来看，很多 Gopher 也都有类似的经历，尤其是那些已经用惯了 `strftime` 方案的 gopher。

下面是一个格式化字符串与实际输出结果的**备忘单**，由 `go-time-operations/timeformat_cheatsheet.go` 生成，可以作为日常在 Go 中进行时间格式化输出的参考。备忘单的第一列为含义，第二列为格式串写法，第三列为对应格式串写法下的输出结果 (取当前时间)：

```shell
2020-06-19 14:44:58 PM +08:00 Jun Fri CST 

Year            | 2006         | 2020         
Year            | 06           | 20           
Month           | 01           | 06           
Month           | 1            | 6            
Month           | Jan          | Jun          
Month           | January      | June         
Day             | 02           | 19           
Day             | 2            | 19           
Week day        | Mon          | Fri          
Week day        | Monday       | Friday       
Hours           | 03           | 02           
Hours           | 3            | 2            
Hours           | 15           | 14           
Minutes         | 04           | 44           
Minutes         | 4            | 44           
Seconds         | 05           | 58           
Seconds         | 5            | 58           
AM or PM        | PM           | PM           
Miliseconds     | .000         | .906         
Microseconds    | .000000      | .906783      
Nanoseconds     | .000000000   | .906783000   
Timezone offset | -0700        | +0800        
Timezone offset | -07:00       | +08:00       
Timezone offset | Z0700        | +0800        
Timezone offset | Z07:00       | +08:00       
Timezone        | MST          | CST          
--------------- + ------------ + ------------ 
```

## 3. 定时器的使用

`time` 包的一个重要功用就是为 Go 开发者提供了**定时器**的实现。`time` 包提供了两类定时器：一次性定时器 `Timer` 和重复定时器 `Ticker`。顾名思义，`Timer` 只能进行一次定时器到期事件触发 (`fire`)，而 Ticker 则可以按一定时间间隔多次触发定时器到期事件。下面我们以一次性定时器 `Timer` 为例，看看如何使用 `time` 包提供的定时器。

### 1) `Timer` 的创建

`time` 包提供了多种创建 `Timer` 定时器的方式，我们通过一个示例来看一下：

```go
// go-time-operations/timer_create.go
package main

import (
	"fmt"
	"time"
)

func create_timer_by_afterfunc() {
	_ = time.AfterFunc(1*time.Second, func() {
		fmt.Println("timer created by afterfunc fired!")
	})
}

func create_timer_by_newtimer() {
	timer := time.NewTimer(2 * time.Second)
	select {
	case <-timer.C:
		fmt.Println("timer created by newtimer fired!")
	}
}

func create_timer_by_after() {
	select {
	case <-time.After(2 * time.Second):
		fmt.Println("timer created by after fired!")
	}
}

func main() {
	create_timer_by_afterfunc()
	create_timer_by_newtimer()
	create_timer_by_after()
} 
```



运行该示例：

```
$go run timer_create.go
timer created by afterfunc fired!
timer created by newtimer fired!
timer created by after fired! 
```



从上面示例中，我们看到了 `Timer` 的三种创建姿势：`NewTimer`、`AfterFunc` 和 `After`。虽然创建姿势稍有差别，但殊途同归，它们背后的原理都是一样的，见下面这幅 `Timer` 创建以及触发原理的示意图：
![37 time包，你用对了吗](https://img-hello-world.oss-cn-beijing.aliyuncs.com/6b3d804bd830401345a607877984cdc4.png)

图 9-4-4：Timer 创建与触发

从图中，我们看到无论采用哪种方式创建 `Timer`，其实质都是在用户层实例化一个 `time.Timer` 结构体：

```go
// $GOROOT/src/time/sleep.go (go 1.14)
type Timer struct {
        C <-chan Time
        r runtimeTimer
}

func NewTimer(d Duration) *Timer {
        c := make(chan Time, 1)
        t := &Timer{
                C: c,
                r: runtimeTimer{
                        when: when(d),
                        f:    sendTime,
                        arg:  c,
                },
        }
        startTimer(&t.r)
        return t
} 
```

该结构体包含两个字段 `C` 和 `r`，其中 `C` 是用户层用户接收定时器触发事件的 Channel，而 `r` 则是一个与 `runtime.timer`(`runtime/time.go`) 对应 (且要保持一致) 的结构。另外我们注意到：`NewTimer` 创建的用于接收定时器触发事件的 channel 是一个带缓冲的 channel。

被实例化后的 `Timer` 将交给运行时层的 `startTimer` 函数，后者使用其初始化运行时层面的 `runtime.timer` 结构，并将 `runtime.timer` 加入到为每个 P 分配的**定时器最小堆**中进行管理。

老版本 Go 中 (Go 1.9 版本之前)，运行时维护一个由互斥锁保护的全局最小堆 (minheap)，定时器最小堆的维护操作都要对其互斥锁进行加解锁操作，导致其性能和伸缩性很差。最新的定时器管理调度方案 (Go 1.14) 抛弃了全局唯一最小堆方案，而是为每个 P (`runtime.p`) 创建一个定时器最小堆并通过网络轮询器 (`net poller`) 在运行时调度的协助下统一对各个定时器最小堆进行管理和调度：

```go
// $GOROOT/src/runtime/runtime2.go (go 1.14)

type p struct {
	... ...
	timersLock mutex
	timers []*timer
	... ... 
```

当运行时调度时发现某个定时器的时间已到，就会将该定时器从其所在最小堆中移除，并在 `runtime.runOneTimer` 中调用相应 `runtime.timer` 的触发函数 `f`。在前面 `NewTimer` 中我们看到这个 `f` 被赋值为 `time.sendTime`：

```go
// $GOROOT/src/time/sleep.go (go 1.14)

func sendTime(c interface{}, seq uintptr) {
        select {
        case c.(chan Time) <- Now():
        default:
        }
} 
```

之前我们提到过 `time.Timer.C` 是一个带缓冲的 channel，目的就是防止运行时在执行 `sendTime` 时被阻塞在该 channel 上。我们看到 `sendTime` 还加了双保险：通过一个 `select` 判断 `channel c` 的缓冲区是否已满，一旦满了，直接退出 (走 default 分支)。一旦 `sendTime` 成功将数据写入 `time.Timer.C`，用户层的代码就会收到该定时器触发事件了。

### 2) `Timer` 的资源释放

很多 Go 初学者在使用 `Timer` 时都会担忧 Timer 的创建会占用系统资源。但我们从上面的 `Timer` 创建和触发原理来看，Go 中的定时器是在 Go 运行时层面实现的，并不会占用系统资源。尤其是新版本定时器的管理和调度已经与运行时网络轮询器 (net poller) 融合在一起了，一个定时器占用的资源仅限于对应数据结构占用的内存以及一个带缓冲 Channel (通过 `AfterFunc` 创建的定时器还会启动一个额外的 goroutine 用于执行用户传入的函数)。当定时器被从最小堆移除并触发事件后，其占用的内存资源、Channel 等都会在后续被垃圾收集器 (GC) 回收掉。

不过即便是再好的方案，在面对大量的定时器时依然会有达到瓶颈的时候，因此作为 `Timer` 的使用者，我们要做的就是尽量减少在使用 `Timer` 时对最小堆管理和垃圾回收的压力，即：**及时调用定时器的 `Stop` 方法从最小堆删除定时器或重用 (`Reset`) 处于活跃状态 (active) 的定时器。**

### 3) 停止 `Timer`

`Timer` 提供了 `Stop` 方法用于将尚未触发的定时器从 P 中的最小堆中移除，使之失效，这样可以减小最小堆管理和垃圾回收的压力。因此，使用定时器时及时调用 `Stop` 方法是一个很好的 Go 语言实践。我们来看一个例子：

```go
// go-time-operations/timer_stop.go 
... ...
func consume(c <-chan bool) bool {
	timer := time.NewTimer(time.Second * 5)
	defer timer.Stop()
	select {
	case b := <-c:
		if b == false {
			log.Printf("recv false, continue")
			return true
		}
		log.Printf("recv true, return")
		return false
	case <-timer.C:
		log.Printf("timer expired")
		return true
	}
}

func main() {
	c := make(chan bool)
	var wg sync.WaitGroup
	wg.Add(2)

	// producer goroutine
	go func() {
		for i := 0; i < 5; i++ {
			time.Sleep(time.Second * 1)
			c <- false
		}

		time.Sleep(time.Second * 1)
		c <- true
		wg.Done()
	}()

	go func() {
		for {
			if b := consume(c); !b {
				wg.Done()
				return
			}
		}
	}()

	wg.Wait()
} 
```

这个例子是个很典型的生产者 - 消费者例子，我们启动两个 goroutine 分别代表生产者和消费者。生成者每隔 1 秒发送一个布尔值给消费者。生产者通过布尔值为 `true` 的消息通知消费者自己已经停止生产退出了。消费者始终处于一个消费循环中，直到收到布尔值为 `true` 的消息才会退出。消费者每次消费时都会启动一个定时器以将每次接收消息等待的时间约束在 5 秒之内，一旦触发超时将重新进行新一轮的消息接收。

如果我们不及时调用 `Timer.Stop` 方法，那么前五轮启动的定时器都将存放在运行时层的最小堆中 (尚未到触发时间)，这显然不是我们期望看到的，于是我们通过 `defer` 实现函数返回即停止定时器，这样一来运行时层最小堆中仅存放我们正在使用的定时器。

运行该示例：

```
$go run timer_stop.go
2020/06/20 19:54:43 recv false, continue
2020/06/20 19:54:44 recv false, continue
2020/06/20 19:54:45 recv false, continue
2020/06/20 19:54:46 recv false, continue
2020/06/20 19:54:47 recv false, continue
2020/06/20 19:54:48 recv true, return 
```

### 4) 重用 `Timer`

在上面例子中，即便我们及时调用了 `Stop` 方法让尚未触发的定时器失效，但我们依旧要创建多个定时器。如果我们不想重复创建这么多 `Timer` 实例，而是重用现有的 `Timer` 实例，那么我们就要用到 `Timer` 的 `Reset` 方法。Go 官方文档建议只对如下两种定时器调用 `Reset` 方法：

- 已经停止了的定时器 (`Stopped`)；
- 已经触发过且 `Timer.C` 中的数据已经被读空。

Go 官方文档还给出了推荐的使用模式：

```go
if !t.Stop() {
        <-t.C
}
t.Reset(d) 
```

接下来，我们就将上面例子改造为只使用一个定时器。

```go
// go-time-operations/timer_reset_1.go
package main

import (
	"log"
	"sync"
	"time"
)

func consume(c <-chan bool, timer *time.Timer) bool {
	if !timer.Stop() {
		<-timer.C
	}
	timer.Reset(5 * time.Second)

	select {
	case b := <-c:
		if b == false {
			log.Printf("recv false, continue")
			return true
		}
		log.Printf("recv true, return")
		return false
	case <-timer.C:
		log.Printf("timer expired")
		return true
	}
}

func main() {
	c := make(chan bool)
	var wg sync.WaitGroup
	wg.Add(2)

	go func() {
		for i := 0; i < 5; i++ {
			time.Sleep(time.Second * 1)
			c <- false
		}

		time.Sleep(time.Second * 1)
		c <- true
		wg.Done()
	}()

	go func() {
		timer := time.NewTimer(time.Second * 5)
		for {
			if b := consume(c, timer); !b {
				wg.Done()
				return
			}
		}
	}()

	wg.Wait()
} 
```

使用 `Reset` 改造后的代码中生产者的行为并未改变，在实际执行时每次循环中，定时器在被重置 (`Reset`) 之前都没有触发 (fire)，因此 `timer.Stop` 的调用均返回 `true`，即成功将 timer 停止。该示例的执行结果如下：

```
$go run timer_reset_1.go
2020/06/21 05:10:20 recv false, continue
2020/06/21 05:10:21 recv false, continue
2020/06/21 05:10:22 recv false, continue
2020/06/21 05:10:23 recv false, continue
2020/06/21 05:10:24 recv false, continue
2020/06/21 05:10:25 recv true, return 
```



我们看到这个输出结果与前面使用 `Stop` 的示例相比并无二致。

现在我们来改变一下生产者的发送行为：从之前每隔 1 秒 “生产” 一次数据变成每隔 7 秒 “生产” 一次数据，而消费者的行为不变。考虑到篇幅，这里仅列出变化的生产者的代码：

```go
// go-time-operations/timer_reset_2.go
... ...
func main() {
	c := make(chan bool)
	var wg sync.WaitGroup
	wg.Add(2)

	go func() {
		for i := 0; i < 5; i++ {
			time.Sleep(time.Second * 7)
			c <- false
		}

		time.Sleep(time.Second * 7)
		c <- true
		wg.Done()
	}()
	... ...
} 
```

我们来看看生产者行为变更后的执行结果：

```
$go run timer_reset_2.go 
2020/06/21 05:14:23 timer expired
fatal error: all goroutines are asleep - deadlock!
... ... 
```

我们看到这次运行的程序**死锁**住了！为什么会出现这种情况呢？我们来分析一下。由于生产者的 “生产” 行为发生了变化，导致消费者在收到第一个数据前有了一次定时器触发 (对应上面输出结果的第一行)，For 循环重启一轮接收。这时 `timer.Stop` 方法返回的不再是 `true` 而是 `false`，因为这个将被重用的 `timer` 已经触发过。于是按照预定逻辑，消费者将尝试抽干 (drain)`timer.C` 中的数据，但 `timer.C` 中此时并没有数据，于是消费者 goroutine 就会阻塞在对该 channel 的读取操作上。而此时生产者处于 `sleep` 状态，主 goroutine 处于 `wait` 状态，Go 运行时判断所有 goroutine 均不能前进，于是报了 `deadlock` 错误。

我们看到问题的根源在于已经触发且其对应的 Channel 已经被取空的 `timer` 已经符合了直接使用 `Reset` 的前提，但我们仍然尝试去抽干 (drain) 该定时器的 Channel，导致消费者 goroutine 阻塞。我们来改进一下该示例：在 `timer.C` 无数据可读的情况下，也不要阻塞在这个 channel 上面：

```
// go-time-operations/timer_reset_3.go
func consume(c <-chan bool, timer *time.Timer) bool {
	if !timer.Stop() {
		select {
		case <-timer.C:
		default:
		}
	}
	timer.Reset(5 * time.Second)

	select {
	case b := <-c:
		if b == false {
			log.Printf("recv false, continue")
			return true
		}
		log.Printf("recv true, return")
		return false
	case <-timer.C:
		log.Printf("timer expired")
		return true
	}
} 
```

在上面改进版示例中，我们使用了一个小技巧：我们通过带有 `default` 分支的 `select` 来处理 `timer.C`，这样当 `timer.C` 中无数据时，代码可以通过 `default` 分支继续向下处理，而不会再阻塞在对 `timer.C` 的读取上了。我们看看运行结果：

```go
$go run timer_reset_3.go 
2020/06/21 05:40:51 timer expired
2020/06/21 05:40:53 recv false, continue
2020/06/21 05:40:58 timer expired
2020/06/21 05:41:00 recv false, continue
2020/06/21 05:41:05 timer expired
2020/06/21 05:41:07 recv false, continue
2020/06/21 05:41:12 timer expired
2020/06/21 05:41:14 recv false, continue
2020/06/21 05:41:19 timer expired
2020/06/21 05:41:21 recv false, continue
2020/06/21 05:41:26 timer expired
2020/06/21 05:41:28 recv true, return 
```

### 5) 重用 `Timer` 时存在的竞态条件

当一个定时器触发时，运行时会调用 `runtime.runOneTimer` 调用定时器关联的触发函数：

```
// $GOROOT/src/runtime/time.go (go 1.14)

func runOneTimer(pp *p, t *timer, now int64) {
	... ...
        unlock(&pp.timersLock)

        f(arg, seq)

        lock(&pp.timersLock)
	... ...
} 
```

我们看到在 `runOneTimer` 执行 `f(arg, seq)` 这个函数前，`runOneTimer` 对 p 的 `timersLock` 进行了解锁操作，也就是说 `f` 的执行并没有在锁内。 `f` 的执行是什么呢？

- 对于通过 `AfterFunc` 创建的定时器来说，就是启动一个新 goroutine，并在这个新 goroutine 中执行用户传入的函数；
- 对于通过 `After` 或 `NewTimer` 创建的定时器而言，`f` 的执行就是 `time.sendTime` 函数，也就是将当前时间写入定时器的通知 Channel 中。

这个时候会有一个竞态条件出现：定时器触发 (fire) 的过程中 `f` 函数的执行与用户层重置定时器前的抽干 Channel 的操作是分别在两个 goroutine 中执行的，谁先谁后，完全依靠运行时调度。于是 `timer_reset_3.go` 中的看似没有问题的代码，也可能存在问题（当然需要时间粒度足够小，比如：毫秒级的定时器）。以通过 `After` 或 `NewTimer` 创建的定时器为例 (即 `f` 函数为 `time.sendTime`)：

- 如果 `sendTime` 的执行发生在抽干 channel 动作之前，那么就是 `timer_reset_3.go` 中的执行结果：`Stop` 方法返回 `false`（因为定时器已经触发了），显式抽干 channel 的动作是可以读出数据的。后续定时器 `Reset` 后，定时器将继续正常运行；
- 如果 `sendTime` 的执行发生在抽干 channel 动作之后，那么问题就来了！虽然 `Stop` 方法返回 `false`（因为定时器已经触发了），但抽干 channel 的动作并没有读出任何数据。之后，`sendTime` 将数据写到 channel 中。这样定时器重置后的定时器 Channel 中实际上已经有了数据，于是当消费者进入下面的 `select` 语句中时，`case <-timer.C` 这一分支因有数据而被直接选中了，没有起到超时等待的作用。也就是说[定时器被重置之后居然又立即 “触发” 了](https://github.com/golang/go/issues/11513)。

目前这个竞态问题尚未有理想解决方案，不过大多数情况下按照 `timer_reset_3.go` 中 `Reset` 的使用方法都是可以正常工作的。

## 4. 小结

本节要点：

- 掌握 time 包表示即时时间的原理 (time.Time)；
- 了解挂钟时间与单调时间的差别；
- 掌握时间基本操作的方法：获取特定时区的当前时间、时间的比较与运算；
- 掌握 Go 特有的时间格式化方法；
- 掌握 Go 定时器的使用和注意事项。