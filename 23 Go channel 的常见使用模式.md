23 Go channel 的常见使用模式

## Go channel 的常见使用模式

**channel**是 Go 语言提供的一种重要并发原语。从前面的章节中我们了解了它在 Go 语言的“Communicating Sequential Processes”并发模型中扮演着重要的角色：既可以用来实现 goroutine 间的通信，同时还可以实现 goroutine 间的同步。

在 Go 中，channel 类型为一等公民（first-class citizen），这意味着我们可以像普通变量那样使用 channel，比如：定义 channel 类型变量、给 channel 变量赋值、将 channel 作为参数传递给函数/方法、将 channel 作为返回值从函数/方法中返回，甚至将 channel 发送到其他 channel 中。

正是由于 channel 一等公民的特性，channel 原语使用起来很简单：

```go
c := make(chan int)    // 创建一个无缓冲(unbuffered)的int类型的channel

c := make(chan int, 5) // 创建一个带缓冲的int类型的Channel

c <- x        // 向channel c中发送一个值
<- c          // 从channel c中接收一个值
x = <- c      // 从channel c接收一个值并将其存储到变量x中
x, ok = <- c  // 从channel c接收一个值。如果channel关闭了，那么ok将被置为false
for i := range c { ... ... } // for range与channel结合使用
close(c)      // 关闭channel c

c := make(chan chan int) // 创建一个无缓冲的chan int类型的channel
func stream(ctx context.Context, out chan<- Value) error // 将只发送(send-only) channel作为函数参数
func spawn(...) <-chan T	// 将只接收(receive-only)类型channel作为返回值 
```

当涉及同时对多个 channel 进行操作时，我们会结合使用到 Go 为 CSP 并发模型提供的另外一个原语：**select**。通过 select，我们可以同时在多个 channel 上进行发送/接收操作：

```go
select {
case x := <-c1: // 从channel c1接收数据
	... ...

case y, ok := <-c2: // 从channel c2接收数据，并根据ok值判断c2是否已经关闭
	... ...

case c3 <- z: // 将z值发送到channel c3中:
	... ...

default: // 当上面case中的channel通信均无法实施时，执行该默认分支
} 
```

我们看到，channel 和 select 两种原语的操作十分简单，这遵循了 Go 语言 **“追求简单”** 的设计哲学，但它们却为 Go 并发程序带来了强大的表达能力。本节我们就来看看 Go 并发原语 channel 的妙用（结合 select）。

## 1. 无缓冲(unbuffered)channel

无缓冲 channel 兼具通信和同步特性，在并发程序中应用颇为广泛。我们可以通过不带有`capacity`参数的内置 make 函数创建一个可用的无缓冲 channel：

```go
c := make(chan T) // T为channel中元素的类型 
```

由于无缓冲 channel 的运行时层实现不带有缓冲区，因此对无缓冲 channel 的接收和发送操作是同步的，即对同一个无缓冲 channel，只有同时进行接收操作的 goroutine 和对其进行发送操作的 goroutine 都存在的情况下，通信才能得以进行，否则单方面的操作会让对应的 goroutine 陷入挂起状态。

- 如果一个无缓冲 channel 没有任何 goroutine 对其进行接收操作，一旦此时有 goroutine 先对其进行发送操作，那么动作发生和完成的时序如下：

```
发送动作发生 -> 接收动作发生(有goroutine对其进行接收操作) -> 发送动作完成/接收动作完成(先后顺序不能确定) 
```

- 如果一个无缓冲 channel 没有任何 goroutine 对其进行发送操作，一旦此时有 goroutine 先对其进行接收操作，那么动作发生和完成的时序如下：

```
接收动作发生 -> 发送动作发生(有goroutine对其进行发送操作) -> 发送动作完成/接收动作完成(先后顺序不确定) 
```

因此，根据上述时序结果，对于无缓冲 channel 而言，我们得到以下结论：

- 发送动作一定发生在接收动作完成之前；
- 接收动作一定发生在发送动作完成之前。

这与 Go 官方[“Go 内存模型”](https://tip.golang.org/ref/mem)一文中对 channel 通信的描述是一致的。也正因为如此，下面的代码可以保证`main`输出的变量 a 的值为`"hello, world"`，因为函数 f 中的 channel 接收动作发生在主 goroutine 对 channel 发送动作完成之前，而`a = "hello, world"`语句又发生在 channel 接收动作之前，因此主 goroutine 在 channel 发送操作完成后看到的变量 a 的值一定是"hello, world"，而不是空字符串。

```go
// go-channel-case-1.go
package main
  
import "time"

var c = make(chan int)
var a string

func f() {
        a = "hello, world"
        <-c
}

func main() {
        go f()
        c <- 5
        println(a)
} 
```

### 1) 用作信号传递

在“Go 并发模型和常见并发模式”一节中，我们已经接触到将 channel 作为信号传递通道的场景，这里再系统梳理一下。

#### a) 1 对 1 通知信号

无缓冲 channel 常被用于在两个 goroutine 之间 1 对 1 的传递通知信号，比如下面这个例子：

```go
// go-channel-case-2.go
package main

import (
	"fmt"
	"time"
)

type signal struct{}

func worker() {
	println("worker is working...")
	time.Sleep(1 * time.Second)
}

func spawn(f func()) <-chan signal {
	c := make(chan signal)
	go func() {
		println("worker start to work...")
		f()
		c <- signal(struct{}{})
	}()
	return c
}

func main() {
	println("start a worker...")
	c := spawn(worker)
	<-c
	fmt.Println("worker work done!")
} 
```

在这个例子中，spawn 函数返回的 channel 被用于承载新 goroutine 退出的**“通知信号”**，该信号专用于通知 main goroutine。main goroutine 在调用 spawn 函数一直阻塞在对这个“通知信号”的接收动作上。

我们来运行一下这个例子：

```shell
$go run go-channel-case-2.go
start a worker...
worker start to work...
worker is working...
worker work done! 
```

#### b) 1 对 n 通知信号

有些时候，无缓冲 channel 还被用来实现**1 对 n 的信号通知**机制。这样的信号通知机制常被用于协调多个 goroutine 一起工作，比如下面的例子：

```go
// go-channel-case-3.go 
package main

import (
	"fmt"
	"sync"
	"time"
)

type signal struct{}

func worker(i int) {
	fmt.Printf("worker %d: is working...\n", i)
	time.Sleep(1 * time.Second)
	fmt.Printf("worker %d: works done\n", i)
}

func spawnGroup(f func(i int), num int, groupSignal <-chan signal) <-chan signal {
	c := make(chan signal)
	var wg sync.WaitGroup

	for i := 0; i < num; i++ {
		wg.Add(1)
		go func(i int) {
			defer wg.Done()
			<-groupSignal
			fmt.Printf("worker %d: start to work...\n", i)
			f(i)
		}(i + 1)
	}

	go func() {
		wg.Wait()
		c <- signal(struct{}{})
	}()
	return c
}

func main() {
	fmt.Println("start a group of workers...")
	groupSignal := make(chan signal)
	c := spawnGroup(worker, 5, groupSignal)
	time.Sleep(5 * time.Second)
	fmt.Println("the group of workers start to work...")
	close(groupSignal)
	<-c
	fmt.Println("the group of workers work done!")
} 
```

在上面例子中，main goroutine 创建了一组 5 个 worker goroutine，这些 goroutine 启动后会阻塞在名为 groupSignal 的无缓冲 channel 上。main goroutine 通过`close(groupSignal)`向所有 worker goroutine 广播出“开始工作”的信号，所有 worker goroutine 在收到 groupSignal 后**“一起”**开始工作，就像起跑线上的运动员听到了裁判员发出的起跑信号枪声。

这个例子的运行结果如下：

```shell
$go run go-channel-case-3.go
start a group of workers...
the group of workers start to work...
worker 3: start to work...
worker 3: is working...
worker 4: start to work...
worker 4: is working...
worker 1: start to work...
worker 1: is working...
worker 5: start to work...
worker 5: is working...
worker 2: start to work...
worker 2: is working...
worker 3: works done
worker 4: works done
worker 5: works done
worker 1: works done
worker 2: works done
the group of workers work done! 
```

我们看到：关闭一个无缓冲 channel 会让所有阻塞在该 channel 上的接收操作返回，从而实现一种 1 对 n 的**“广播”**机制。该 1 对 n 的信号通知机制还常用于通知一组 worker goroutine 退出，比如下面例子：

```go
// go-channel-case-4.go
package main

import (
	"fmt"
	"sync"
	"time"
)

type signal struct{}

func worker(i int, quit <-chan signal) {
	fmt.Printf("worker %d: is working...\n", i)
LOOP:
	for {
		select {
		default:
			// 模拟worker工作
			time.Sleep(1 * time.Second)

		case <-quit:
			break LOOP
		}
	}
	fmt.Printf("worker %d: works done\n", i)
}

func spawnGroup(f func(int, <-chan signal), num int, groupSignal <-chan signal) <-chan signal {
	c := make(chan signal)
	var wg sync.WaitGroup

	for i := 0; i < num; i++ {
		wg.Add(1)
		go func(i int) {
             defer wg.Done()
			fmt.Printf("worker %d: start to work...\n", i)
			f(i, groupSignal)
		}(i + 1)
	}

	go func() {
		wg.Wait()
		c <- signal(struct{}{})
	}()
	return c
}

func main() {
	fmt.Println("start a group of workers...")
	groupSignal := make(chan signal)
	c := spawnGroup(worker, 5, groupSignal)
	fmt.Println("the group of workers start to work...")

	time.Sleep(5 * time.Second)

	// 通知workers退出
	fmt.Println("notify the group of workers to exit...")
	close(groupSignal)
	<-c
	fmt.Println("the group of workers work done!")
} 
```

运行该示例：

```shell
$go run go-channel-case-4.go
start a group of workers...
the group of workers start to work...
worker 1: start to work...
worker 1: is working...
worker 3: start to work...
worker 3: is working...
worker 5: start to work...
worker 5: is working...
worker 4: start to work...
worker 4: is working...
worker 2: start to work...
worker 2: is working...
notify the group of workers to exit...
worker 2: works done
worker 4: works done
worker 5: works done
worker 1: works done
worker 3: works done
the group of workers work done! 
```

### 2) 用于替代锁机制

无缓冲 channel 具有同步特性，这让它在某些场合可以替代锁，从而使得程序更加清晰，可读性更好。下面是一个传统的基于“共享内存”+“锁”模式的 goroutine 安全的计数器的实现：

```go
// go-channel-case-5.go
package main

import (
	"fmt"
	"sync"
	"time"
)

type counter struct {
	sync.Mutex
	i int
}

var cter counter

func Increase() int {
	cter.Lock()
	defer cter.Unlock()
	cter.i++
	return cter.i
}

func main() {
	for i := 0; i < 10; i++ {
		go func(i int) {
			v := Increase()
			fmt.Printf("goroutine-%d: current counter value is %d\n", i, v)
		}(i)
	}

	time.Sleep(5 * time.Second)
} 
```

下面是我们使用无缓冲 channel 替代锁后的实现：

```go
// go-channel-case-6.go
package main

import (
	"fmt"
	"time"
)

type counter struct {
	c chan int
	i int
}

var cter counter

func InitCounter() {
	cter = counter{
		c: make(chan int),
	}

	go func() {
		for {
			cter.i++
			cter.c <- cter.i
		}
	}()
	fmt.Println("counter init ok")
}

func Increase() int {
	return <-cter.c
}

func init() {
	InitCounter()
}

func main() {
	for i := 0; i < 10; i++ {
		go func(i int) {
			v := Increase()
			fmt.Printf("goroutine-%d: current counter value is %d\n", i, v)
		}(i)
	}

	time.Sleep(5 * time.Second)
} 
```

在这个实现中，我们将计数器操作全部交给一个独立的 goroutine 去处理，并通过无缓冲 channel 的同步阻塞特性实现计数器的控制。这样其他 goroutine 通过 Increase 函数试图增加计数器值的动作实质上就转化为一次无缓冲 channel 的接收动作。这种并发设计逻辑更符合 Go 语言所倡导的**“不要通过共享内存来通信，而是通过通信来共享内存”**的原则。

运行该示例，我们得到如下结果：

```shell
$go run go-channel-case-6.go 
counter init ok
goroutine-9: current counter value is 10
goroutine-0: current counter value is 1
goroutine-6: current counter value is 7
goroutine-2: current counter value is 3
goroutine-8: current counter value is 9
goroutine-4: current counter value is 5
goroutine-5: current counter value is 6
goroutine-1: current counter value is 2
goroutine-7: current counter value is 8
goroutine-3: current counter value is 4 
```



## 2. 带缓冲(buffered)channel

和无缓冲 channel 不同，我们通过带有`capacity`参数的内置 make 函数可以创建一个可用的带缓冲 channel：

```go
c := make(chan T, capacity) // T为channel中元素的类型, capacity为带缓冲channel的缓冲区容量 
```

由于带缓冲 channel 的运行时层实现带有缓冲区，因此对带缓冲 channel 的发送操作在缓冲区未满、接收操作在缓冲区非空的情况下是**异步**的(发送或接收无需阻塞等待)。即对一个带缓冲 channel，在缓冲区未满的情况下，对其进行发送操作的 goroutine 不会阻塞挂起；在缓冲区有数据的情况下，对其进行接收操作的 goroutine 亦不会阻塞挂起。但当缓冲区满的情况下，对其进行发送操作的 goroutine 会阻塞挂起；当缓冲区未空的情况下，对其进行接收操作的 goroutine 亦会阻塞挂起。

### 1) 用作消息队列

channel 经常被 Go 初学者视为在 goroutines 间通信的消息队列，这是由于 channel 的原生特性与我们认知中的消息队列十分相似，包括 goroutine 安全、有 fifo(first-in, first out)保证等。

和无缓冲 channel 更多用于信号/事件管道相比，可自行设置容量、异步收发的带缓冲 channel 更适合被用作为消息队列，并且带缓冲 channel 在数据收发性能上要明显好于无缓冲 channel。下面是一些关于无缓冲 channel 和带缓冲 channel 收发性能测试的结果(Go 1.13.6, MacBook Pro 8 核)：

- 单接收单发送性能基准测试

我们先来看看针对一个 channel 只有一个发送 goroutine 和一个接收 goroutine 的情况下，两种 channel 的收发性能比对数据：

```shell
 // 无缓冲channel
// go-channel-operation-benchmark/unbuffered-chan

$go test -bench . one_to_one_test.go
goos: darwin
goarch: amd64
BenchmarkUnbufferedChan1To1Send-8   	 6202120	       198 ns/op
BenchmarkUnbufferedChan1To1Recv-8   	 6752820	       178 ns/op
PASS
ok  	command-line-arguments	2.811s


// 带缓冲channel
// go-channel-operation-benchmark/buffered-chan

$go test -bench . one_to_one_cap_10_test.go
goos: darwin
goarch: amd64
BenchmarkBufferedChan1To1SendCap10-8   	14397186	        83.7 ns/op
BenchmarkBufferedChan1To1RecvCap10-8   	14275723	        82.2 ns/op
PASS
ok  	command-line-arguments	2.555s

$go test -bench . one_to_one_cap_100_test.go
goos: darwin
goarch: amd64
BenchmarkBufferedChan1To1SendCap100-8  	18011007	        65.5 ns/op
BenchmarkBufferedChan1To1RecvCap100-8  	18031082	        65.4 ns/op
PASS
ok  	command-line-arguments	2.499s 
```

- 多接收多发送性能基准测试

我们再来看看针对一个 channel 有多个发送 goroutine 和多个接收 goroutine 的情况下，两种 channel 的收发性能比对数据(这里建立 10 个发送 goroutine 和 10 个接收 goroutine)：

```shell
// 无缓冲channel
// go-channel-operation-benchmark/unbuffered-chan

$go test -bench . multi_to_multi_test.go 
goos: darwin
goarch: amd64
BenchmarkUnbufferedChanNToNSend-8   	  317324	      3793 ns/op
BenchmarkUnbufferedChanNToNRecv-8   	  295288	      4139 ns/op
PASS
ok  	command-line-arguments	2.516s

// 带缓冲channel
// go-channel-operation-benchmark/buffered-chan

$go test -bench . multi_to_multi_cap_10_test.go 
goos: darwin
goarch: amd64
BenchmarkBufferedChanNToNSendCap10-8   	  534625	      2252 ns/op
BenchmarkBufferedChanNToNRecvCap10-8   	  476221	      2752 ns/op
PASS
ok  	command-line-arguments	2.573s

$go test -bench .  multi_to_multi_cap_100_test.go
goos: darwin
goarch: amd64
BenchmarkBufferedChanNToNSendCap100-8    1000000	      1283 ns/op
BenchmarkBufferedChanNToNRecvCap100-8    1000000	      1250 ns/op
PASS
ok  	command-line-arguments	2.564s 
```

综合以上结果数据，我们可以得到两个结论：

- 无论是 1 收 1 发还是多收多发，带缓冲 channel 的收发性能要好于无缓冲 channel；
- 对于带缓冲 channel 而言，选择适当容量会在一定程度上提升一定收发性能

### 2) 用作计数信号量(counting semaphore)

Go 并发设计的一个惯用法就是将带缓冲 channel 用作计数信号量（counting semaphore）。带缓冲 channel 中的当前数据个数代表的是当前同时处于活动状态（处理业务）的 goroutine 的数量，而带缓冲 channel 的容量（capacity）就代表了允许同时处于活动状态的 goroutine 的最大数量。向带缓冲 channel 的一个发送操作表示获取一个信号量，而从 channel 的一个接收操作则表示释放一个信号量。

下面是一个将带缓冲 channel 用作计数信号量的例子：

```go
// go-channel-case-7.go
package main

import (
	"log"
	"sync"
	"time"
)

var active = make(chan struct{}, 3)
var jobs = make(chan int, 10)

func main() {
	go func() {
		for i := 0; i < 8; i++ {
			jobs <- (i + 1)
		}
		close(jobs)
	}()

	var wg sync.WaitGroup

	for j := range jobs {
		wg.Add(1)
		go func(j int) {
             defer wg.Done()
			active <- struct{}{}
			log.Printf("handle job: %d\n", j)
			time.Sleep(2 * time.Second)
			<-active
		}(j)
	}
	wg.Wait()
} 
```

我们看到：上面的示例创建了一组 goroutines 来处理 job，同一时间允许的最多 3 个 goroutine 处于活动状态。为达成这一目标，我们看到示例使用了一个容量(capacity)为 3 的带缓冲 channel: **active**作为计数信号量，这意味着允许同时处于**活动状态**的最大 goroutine 数量为 3。我们运行一下该示例：

```shell
$go run go-channel-case-7.go 
2020/02/04 09:57:02 handle job: 8
2020/02/04 09:57:02 handle job: 4
2020/02/04 09:57:02 handle job: 1
2020/02/04 09:57:04 handle job: 2
2020/02/04 09:57:04 handle job: 3
2020/02/04 09:57:04 handle job: 7
2020/02/04 09:57:06 handle job: 6
2020/02/04 09:57:06 handle job: 5 
```

从示例运行结果中的时间戳我们可以看到：虽然我们创建了很多 goroutine，但由于计数信号量的存在，同一时间内处理活动状态(正在处理 job)的 goroutine 的数量最多为 3 个。

### 3) len(channel)的应用

**len**是 Go 语言的一个[built-in 函数](https://tip.golang.org/ref/spec#Length_and_capacity)，它支持接受数组、切片、map、字符串和 channel 类型的参数，并返回对应类型的“长度” - 一个整型值。以`len(s)`为例：

- 如果 s 是字符串类型(string)，len(s) 返回字符串中的字节个数；
- 如何 s 是 [n]T 或 *[n]T 的数组类型，len(s) 返回数组的长度 n；
- 如果 s 是 []T 的切片类型(slice)，len(s) 返回切片的当前长度；
- 如果 s 是 map[K]T 的 map 类型，len(s) 返回 map 中的已定义的 key 的个数；
- 如果 s 是 chan T 类型，那么 len(s) 针对 channel 的类型不同，有如下两种语义：
  - 当 s 为无缓冲 channel 时，len(s) 总是返回 0；
  - 当 s 为带缓冲 channel 时，len(s) 返回当前 channel s 中尚未被读取的元素个数。

这样一来，针对带缓冲 channel 的 len 调用似乎才是有意义的。那我们是否可以使用 len 函数来实现带缓冲 channel 的“判满”、“判有”和“判空”逻辑呢，就像下面示例中伪代码这样：

```go
var c chan T = make(chan T, capacity)

// 判空
if len(c) == 0 {
    // 此时channel c空了?
}

// 判有
if len(c) > 0 {
    // 此时channel c有数据?
}

// 判满
if len(channel) == cap(channel) {
    // 此时channel c满了?
} 
```

大家看到我在上面代码注释的“空了”、“有数据”和“满了”的后面**打上了问号**！channel 原语用于多个 goroutine 间的通信，一旦多个 goroutine 共同对 channel 进行收发操作，len(channel)就会在多个 goroutine 间形成“竞态”，单纯地依靠 len(channel)来判断 channel 中元素状态，不能保证在后续对 channel 的收发时 channel 状态是不变的。以判空为例：

![23 Go channel 的常见使用模式](https://img-hello-world.oss-cn-beijing.aliyuncs.com/130fef61ff83f1e9fb981ba6ad57bd2f.png)

图6-3-1：多goroutine收发channel时的竞态

从上图可以看到，当 goroutine1 使用 len(channel)判空后，便尝试从 channel 中接收数据。但在真正从 channel 读数据前，另外一个 goroutine2 已经将数据读了出去，goroutine1 后面的**读取将阻塞在 channel 上**，导致后面逻辑的失效。因此，**为了不阻塞在 channel 上**，常见的方法是将“判空与读取”放在一个“事务”中，将“判满与写入”放在一个“事务”中，而这类“事务”我们可以通过 select 实现。我们来看下面示例：

```go
// go-channel-case-8.go 
package main

import (
	"fmt"
	"time"
)

func producer(c chan<- int) {
	var i int = 1
	for {
		time.Sleep(2 * time.Second)
		ok := trySend(c, i)
		if ok {
			fmt.Printf("[producer]: send [%d] to channel\n", i)
			i++
			continue
		}
		fmt.Printf("[producer]: try send [%d], but channel is full\n", i)
	}
}

func tryRecv(c <-chan int) (int, bool) {
	select {
	case i := <-c:
		return i, true

	default:
		return 0, false
	}
}

func trySend(c chan<- int, i int) bool {
	select {
	case c <- i:
		return true
	default:
		return false
	}
}

func consumer(c <-chan int) {
	for {
		i, ok := tryRecv(c)
		if !ok {
			fmt.Println("[consumer]: try to recv from channel, but the channel is empty")
			time.Sleep(1 * time.Second)
			continue
		}
		fmt.Printf("[consumer]: recv [%d] from channel\n", i)
		if i >= 3 {
			fmt.Println("[consumer]: exit")
			return
		}
	}
}

func main() {
	c := make(chan int, 3)
	go producer(c)
	go consumer(c)

	select {} // 故意阻塞在此
} 
```



我们看到由于用到了 select 原语的 default 分支语义，当 channel 空的时候，tryRecv 不会阻塞；当 channel 满的时候，trySend 也不会阻塞。我们运行一下该示例：

```shell
$go run go-channel-case-8.go              
[consumer]: try to recv from channel, but the channel is empty
[consumer]: try to recv from channel, but the channel is empty
[producer]: send [1] to channel
[consumer]: recv [1] from channel
[consumer]: try to recv from channel, but the channel is empty
[consumer]: try to recv from channel, but the channel is empty
[producer]: send [2] to channel
[consumer]: recv [2] from channel
[consumer]: try to recv from channel, but the channel is empty
[consumer]: try to recv from channel, but the channel is empty
[producer]: send [3] to channel
[consumer]: recv [3] from channel
[consumer]: exit
[producer]: send [4] to channel
[producer]: send [5] to channel
[producer]: send [6] to channel
[producer]: try send [7], but channel is full
[producer]: try send [7], but channel is full
[producer]: try send [7], but channel is full 
```



这种方法可以适合大多数的场合，但是这种方法有一个“问题”，那就是它改变了 channel 的状态：接收了一个元素或发送了一个元素。有些时候我们不想这么做，我们想在不改变 channel 状态的前提下单纯地侦测 channel 的状态而又不会因 channel 满或空阻塞在 channel 上。但很遗憾，目前没有一种方法可以在实现这样的功能的同时又适用于所有场合。但是在特定的场景下，我们可以用 len(channel)来实现。比如下面这两种场景：

![23 Go channel 的常见使用模式](https://img-hello-world.oss-cn-beijing.aliyuncs.com/1738298d812ef46acb084f5ca1cc6ca3.png)

图6-3-2：两种适合使用len(channel)侦测channel状态的的场景

上图中的情景(a)是一个“多发送单接收”的场景，即有多个发送者，但**有且只有一个接收者**。在这样的场景下，我们可以在接收 goroutine 中使用`len(channel)是否大于0`来判断是否 channel 中有数据需要接收；

情景(b)是一个“多接收单发送”的场景，即有多个接收者，但**有且只有一个发送者**。在这样的场景下，我们可以在发送 goroutine 中使用`len(channel)是否小于cap(channel)`来判断是否可以执行向 channel 的发送操作。

### 3. nil channel 的妙用

对一个没有初始化的 channel(nil channel)进行读写操作都将发生阻塞，比如下面这段代码：

```go
func main() {
	var c chan int
	<-c
}

或者
func main() {
	var c chan int
	c<-1
} 
```



上述无论哪段代码被执行，我们将得到类似如下的错误信息：

```shell
fatal error: all goroutines are asleep - deadlock!
goroutine 1 [chan receive (nil chan)]:
或
goroutine 1 [chan send (nil chan)]: 
```

main goroutine 被阻塞在 channel 上，导致 Go 运行时认为“deadlock”状态出现而抛出 panic。

但 nil channel 也不是一无是处，有些时候妙用 nil channel 可以得到事半功倍的效果。我们来看一个例子：

```go
// go-channel-case-9.go 
package main

import "fmt"
import "time"

func main() {
	c1, c2 := make(chan int), make(chan int)
	go func() {
		time.Sleep(time.Second * 5)
		c1 <- 5
		close(c1)
	}()

	go func() {
		time.Sleep(time.Second * 7)
		c2 <- 7
		close(c2)
	}()

	var ok1, ok2 bool
	for {
		select {
		case x := <-c1:
			ok1 = true
			fmt.Println(x)
		case x := <-c2:
			ok2 = true
			fmt.Println(x)
		}

		if ok1 && ok2 {
			break
		}
	}
	fmt.Println("program end")
} 
```

在这个示例中，我们期望程序在接收完 c1 和 c2 两个 channel 上的数据后就退出。但实际的运行情况如下：

```shell
$go run go-channel-case-9.go
5
0
0
0
... ... //循环输出0
7
program end 
```

我们原本期望上述程序在依次输出 5 和 7 两个数字后退出，但实际运行的输出结果却是在输出 5 之后，程序输出了许多的 0 值后才输出 7 并退出。

我们简单分析一下上述代码的运行过程：

- 前 5s，select 一直处于阻塞状态；
- 第 5s，c1 返回一个 5 后被 close，select 语句的`case x := <-c1`这个分支被选出执行，程序输出 5，并回到 for 循环并重新 select；
- 由于 c1 被关闭，从一个已关闭的 channel 接收数据将永远不会被阻塞，于是新一轮 select 又把`case x := <-c1`这个分支选出并执行。由于 c1 处于关闭状态，从这个 channel 获取数据我们会得到该 channel 对应类型的零值，这里就是 0。于是程序再次输出 0；程序安这个逻辑循环执行，一直输出 0 值；
- 2s 后，c2 被写入了一个数值 7；这样在某一轮 select 的过程中，分支`case x := <-c2`被选中得以执行，程序输出 7 之后满足退出条件，于是程序终止。

我们怎么来改进一下这个程序使之能按照我们的预期输出呢？nil channel 是时候登场了！改进后的示例代码如下：

```go
// go-channel-case-10.go 
package main

import "fmt"
import "time"

func main() {
	c1, c2 := make(chan int), make(chan int)
	go func() {
		time.Sleep(time.Second * 5)
		c1 <- 5
		close(c1)
	}()

	go func() {
		time.Sleep(time.Second * 7)
		c2 <- 7
		close(c2)
	}()

	for {
		select {
		case x, ok := <-c1:
			if !ok {
                  // set channel to nil to block it forever if the channel is empty
				c1 = nil
			} else {
				fmt.Println(x)
			}
		case x, ok := <-c2:
			if !ok {
				c2 = nil
			} else {
				fmt.Println(x)
			}
		}
		if c1 == nil && c2 == nil {
			break
		}
	}
	fmt.Println("program end")
} 
```

上面改进后的示例程序的一个最关键的变化是在判断 c1 或 c2 被关闭后，将显式地将 c1 或 c2 置为 nil。我们知道：**对一个 nil channel 执行获取操作，该操作将阻塞**，于是已经被置为 nil 的 c1 或 c2 的分支将再也不会被 select 选中执行。于是上述改进后的示例的运行结果如下：

```shell
$go run go-channel-case-10.go 
5
7
program end 
```

## 4. 与 select 结合使用的一些惯用法

channel 和 select 的结合使用能形成强大的表达能力，我们在前面的例子中已经或多或少见识过了。这里再强调几种与 select 结合的惯用法。

### 1) 利用 default 分支避免阻塞

select 语句的 default 分支的语义就是在其他非 default 分支因通信未就绪而无法被选择的时候执行的，这就给 default 分支赋予了一种“避免阻塞”的特性。其实在前面的**“len(channel)的应用”**小节的例子中，我们就已经用到了“利用 default 分支”实现的`trySend`和`tryRecv`两个函数：

```go
func tryRecv(c <-chan int) (int, bool) {
	select {
	case i := <-c:
		return i, true

	default: // channel为空 avoid to be blocked
		return 0, false
	}
}

func trySend(c chan<- int, i int) bool {
	select {
	case c <- i:
		return true
	default: // channel满了
		return false
	}
} 
```

无论是无缓冲 channel 还是带缓冲 channel，上面两个两个函数均适用，并且不会阻塞在空 channel 或元素个数已经达到容量的 channel 上。在 Go 标准库中，这个惯用法也有应用，比如：

```go
// $GOROOT/src/time/sleep.go
func sendTime(c interface{}, seq uintptr) {
        // 无阻塞的向c发送当前时间
	// ...
        select {
        case c.(chan Time) <- Now():
        default:
        }
} 
```

### 2) 实现超时机制

带超时机制的 select 是 Go 一种常见的 select 和 channel 的组合用法，通过超时事件，我们既可以避免长期陷入某种操作的等待中，也可以做一些异常处理工作。下面示例代码实现了一次具有 30s 超时的 select：

```go
func worker() {
	select {
	case <-c:
	     // ... do some stuff
	case <-time.After(30 *time.Second):
        // 善后操作(release source like close file or cancel request with cancellable ctx in sub goroutine, etc)
	    return
	}
} 
```

应用带有超时机制的 select 时，要特别注意 timer 使用后的释放，尤其在大量创建 timer 时。Go 语言标准库提供的 timer 实质上是由 Go 运行时自行维护的，而不是操作系统级的定时器资源。Go 运行时启动了一个单独的 goroutine，该 goroutine 执行了一个名为`timerproc`的函数，维护了一个“最小堆”。该 goroutine 会定期被唤醒并读取堆顶的 timer 对象，执行该 timer 对象对应的函数(向 timer.C 中发送一条数据，触发定时器)，执行完毕后就会从最小堆中移除该 timer 对象。创建一个 time.Timer 实则就是在这个最小堆中添加一个 timer 对象实例，而调用 timer.Stop 方法则是从堆中删除对应的 timer 对象。

作为 time.Timer 的使用者，我们要做的就是要尽量减少在使用 Timer 时对管理最小堆的 goroutine 和 Go GC 的压力，**即要及时调用 timer 的 Stop 方法从最小堆删除尚未到达过期时间的 timer 对象。**

### 3) 实现心跳机制

结合 time 包的 Ticker，我们可以实现带有心跳机制的 select。这种机制使得我们可以在监听 channel 的同时，执行一些**周期性的任务**，比如下面这段代码：

```go
func worker() {
	heartbeat := time.NewTicker(30 * time.Second)
	defer heartbeat.Stop()
	for {
		select {
		case <-c:
			// ... do some stuff
		case <- heartbeat.C:
			//... do heartbeat stuff
		}
	}
} 
```



和 timer 一样，我们在使用完 ticker 之后，**不要忘记调用其 Stop 方法以关闭心跳事件在 ticker 的 channel(上面示例中的 heartbeat.C)中持续产生。**

## 5. 小结

本节要点：

- 了解 Go 并发原语：channel 和 select 的基本语义；
- 掌握无缓冲 channel 在信号传递、替代锁同步场景下的应用模式；
- 掌握带缓冲 channel 在消息队列、计数信号量场景下的应用模式，了解在特定场景下利用 len 函数侦测带缓冲 channel 的状态；
- 了解 nil channel 在特定场景下的用途；
- 掌握 select 与 channel 结合使用的一些惯用法以及注意事项。