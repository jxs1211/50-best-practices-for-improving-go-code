38 小心被kill！不要忽略对系统信号的处理



## 小心被kill！不要忽略对系统信号的处理

## 1. 为什么不能忽略对系统信号的处理

**系统信号(signal)** 是一种软件中断，它提供了一种异步的事件处理机制，用于操作系统内核或其他应用进程通知某一应用进程发生了某种事件。比如：一个在终端前台启动的程序，当用户在键盘上按下中断键(一般是`ctrl+c`)时，该程序的进程将会收到内核发给它的中断信号(`SIGINT`)。我们用下面示例来说明一下应用进程收到`SIGINT`中断信号后的情况：

```go
// go-signal/go-program-without-signal-handling.go 
... ...
func main() {
	var wg sync.WaitGroup
	errChan := make(chan error, 1)
	http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprintf(w, "Hello, Signal!\n")
	})
	wg.Add(1)
	go func() {
		errChan <- http.ListenAndServe("localhost:8080", nil)
		wg.Done()
	}()

	select {
	case <-time.After(2 * time.Second):
		fmt.Println("web server start ok")
	case err := <-errChan:
		fmt.Println("web server start failed:", err)
	}
	wg.Wait()
	fmt.Println("web server shutdown ok")
} 
```

这是一个`“Hello，World”`级别的HTTP服务示例，我们编译该程序并在终端前台启动该程序：

```
$go build -o httpserv go-program-without-signal-handling.go
$./httpserv 
web server start ok 
```



接下来我们通过键盘按下中断键(`ctrl+c`)，我们发现程序直接退出了，并且我们期望的`“web server shutdown ok”`的程序退出提示并没有出现在终端控制台上。

应用程序收到系统信号后，一般有三种处理信号的方式：

- 执行系统默认处理动作

对于中断键触发的`SIGINT`信号，系统的默认处理动作是终止该应用进程，这也是上面示例采用的信号处理方式，也是上面示例没有输出退出提示就退出了的原因。对于大多数系统信号，系统默认的处理动作都是终止该进程；

- 忽略信号

如果应用选择忽略对某些信号的处理，那么当应用进程收到这些信号后，既不会执行系统默认处理动作，也不会执行其他自定义的处理动作，信号被忽略掉了，就好像该信号从来就没有发生过似的。系统的大多数信号都可使用这种方式进行处理；

- 捕捉信号并执行自定义处理动作

如果应用进程针对某些信号，既不想执行系统默认处理动作，也不想忽略信号，那么它可以预先**提供一个包含自定义处理动作的函数**，并告知系统当接收到某些信号时调用这个函数。**系统中有两个系统信号是不能被捕捉的，它们是终止程序信号`SIGKILL`和挂起程序信号`SIGSTOP`。**

对于服务端程序而言，一般都是以守护进程(`daemon`)的形式运行在后台的并且我们一般都是通过**系统信号**通知这些守护程序执行退出操作的。在这样的情况下，如果我们选择以系统默认处理方式处理这些退出通知信号，那么守护进程将会被直接杀死，没有任何机会执行一些清理和收尾工作，比如：等待尚未处理完的事务执行完毕、将未保存的数据强制落盘、将某些尚未处理的消息序列化到磁盘(等下次启动后处理)等。这将导致某些处理过程被强制中断而丢失消息，留下无法恢复的现场，导致消息被破坏，甚至会影响下次应用的启动运行。

因此，对于运行在生产环境下的程序，**我们不要忽略对系统信号的处理**，我们应**采用捕捉退出信号的方式**执行自定义的收尾处理函数。

## 2. Go语言对系统信号处理的支持

信号机制的历史久远，早在最初的`Unix`系统版本上就能看到它的身影。信号机制也一直在演化，从最初的**不可靠信号机制**到后来的**可靠信号机制**，直到POSIX.1将其标准化后，系统信号机制才稳定下来，但各个平台对信号机制的支持仍有差异。我们可以通过`kill -l`命令查看各个系统对信号的支持情况：

```shell
// ubuntu 18.04 
$kill -l
 1) SIGHUP	 2) SIGINT	 3) SIGQUIT	 4) SIGILL	 5) SIGTRAP
 6) SIGABRT	 7) SIGBUS	 8) SIGFPE	 9) SIGKILL	10) SIGUSR1
11) SIGSEGV	12) SIGUSR2	13) SIGPIPE	14) SIGALRM	15) SIGTERM
16) SIGSTKFLT	17) SIGCHLD	18) SIGCONT	19) SIGSTOP	20) SIGTSTP
21) SIGTTIN	22) SIGTTOU	23) SIGURG	24) SIGXCPU	25) SIGXFSZ
26) SIGVTALRM	27) SIGPROF	28) SIGWINCH	29) SIGIO	30) SIGPWR
31) SIGSYS	34) SIGRTMIN	35) SIGRTMIN+1	36) SIGRTMIN+2	37) SIGRTMIN+3
38) SIGRTMIN+4	39) SIGRTMIN+5	40) SIGRTMIN+6	41) SIGRTMIN+7	42) SIGRTMIN+8
43) SIGRTMIN+9	44) SIGRTMIN+10	45) SIGRTMIN+11	46) SIGRTMIN+12	47) SIGRTMIN+13
48) SIGRTMIN+14	49) SIGRTMIN+15	50) SIGRTMAX-14	51) SIGRTMAX-13	52) SIGRTMAX-12
53) SIGRTMAX-11	54) SIGRTMAX-10	55) SIGRTMAX-9	56) SIGRTMAX-8	57) SIGRTMAX-7
58) SIGRTMAX-6	59) SIGRTMAX-5	60) SIGRTMAX-4	61) SIGRTMAX-3	62) SIGRTMAX-2
63) SIGRTMAX-1	64) SIGRTMAX	

// macOS 10.14.6
$kill -l
HUP INT QUIT ILL TRAP ABRT EMT FPE KILL BUS SEGV SYS PIPE ALRM TERM URG STOP TSTP CONT CHLD TTIN TTOU IO XCPU XFSZ VTALRM PROF WINCH INFO USR1 USR2 
```

我们看到`kill -l`列出了每个平台支持的信号的列表，其中每个信号都包含信号名称(`signal name`，比如：`SIGINT`)和信号编号(`signal number`，比如：`SIGINT`的编号是2)。

使用kill命令，我们可以将特定信号(通过信号名称或信号编号)发送给某应用进程：

```shell
$kill -s signal_name pid // 如：kill -s SIGINT 20023
$kill -signal_number pid // 如：kill -2 20023 
```

信号机制经过多年演化，已经变得十分复杂繁琐（考虑多种平台对标准的支持程度不一），诸如： 不可靠信号、可靠信号、阻塞信号、信号处理函数的可重入等。如果让开发人员自己来处理这些复杂性，那么势必是一份不小的心智负担。Go语言将这些复杂性留给了运行时层，给用户层提供了体验相当友好接口：`os/signal`包。

Go语言在标准库的`os/signal`包中提供了五个函数(截至Go 1.14版本)供gopher们使用，这里面最主要的函数是`Notify`函数：

```go
func Notify(c chan<- os.Signal, sig ...os.Signal) 
```

该函数用来设置捕捉那些应用关注的系统信号，并在Go运行时层与Go用户层之间用一个channel相连。Go运行时捕捉到应用关注的信号后，会将信号写入channel，这样监听该channel的用户层代码便可以收到该信号通知。我们用一副图来直观看一下Go运行时进行系统信号处理以及与用户层交互的原理：

![38 小心被kill！不要忽略对系统信号的处理](https://img-hello-world.oss-cn-beijing.aliyuncs.com/7d8305aeb9076a52d29cfafe853ec01d.png)

图9-5-1：Go运行时处理信号的原理

在这幅图中，我们看到了Go运行时与用户层有两个“交互点”，一个是上面所说的承载信号交互的channel，而另一个则是运行时层引发的`panic`。

这里Go将信号分为两大类，一类是同步信号(`synchronous signal`)，另外一类是异步信号(`asynchronous signal`)：

- 同步信号

同步信号是指那些因程序执行错误引发的信号，包括：`SIGBUS(总线错误/硬件异常)`、`SIGFPE(算术异常)`和`SIGSEGV(段错误/无效内存引用)`这三个信号。一旦应用进程中的Go运行时收到这三个信号，意味着应用极大可能出现了严重bug，无法继续执行下去，这时Go运行时不会简单地将信号通过channel发送到用户层并等待用户层的异步处理，而是直接将信号转换成一个运行时`panic`并抛出。如果用户层没有专门的`panic`恢复代码，那么Go应用将默认异常退出。

- 异步信号

除了上述的同步信号，其余信号都被Go划归为异步信号。异步信号不是由程序执行错误引起的，而是由其他程序或操作系统内核发出的。异步信号的默认处理行为是因信号而异的。 

`SIGHUP`、`SIGINT`和`SIGTERM` 这三个信号将导致程序直接退出；

`SIGQUIT`、`SIGILL`、`SIGTRAP`、`SIGABRT`、`SIGSTKFLT`、`SIGEMT`和`SIGSYS`在导致程序退出的同时，还会将程序退出时的栈状态打印出来；

`SIGPROF`信号则是被Go运行时用于实现运行时CPU性能剖析指标采集。

其他信号不常用，均采用操作系统的默认处理动作。对于用户层通过`Notify`函数捕获的信号，Go运行时则通过channel将信号发给用户层。

到这里，我们知道了Notify无法捕捉`SIGKILL`和`SIGSTOP`(机制决定的)，也无法捕捉同步信号(Go运行时决定的)，只有捕捉异步信号才是有意义的。下面的例子直观展示了无法被捕获的信号、同步信号以及异步信号的运作机制：

```go
// go-signal/go-program-notify-sync-and-async-signal.go 
package main

import (
	"fmt"
	"os"
	"os/signal"
	"sync"
	"syscall"
	"time"
)

func catchAsyncSignal(c chan os.Signal) {
	for {
		s := <-c
		fmt.Println("收到异步信号:", s)
	}
}

func triggerSyncSignal() {
	time.Sleep(3 * time.Second)
	defer func() {
		if e := recover(); e != nil {
			fmt.Println("恢复panic:", e)
			return
		}
	}()

	var a, b = 1, 0
	fmt.Println(a / b)
}

func main() {
	var wg sync.WaitGroup
	c := make(chan os.Signal, 1)
	signal.Notify(c, syscall.SIGFPE,
		syscall.SIGINT,
		syscall.SIGKILL)

	wg.Add(2)
	go func() {
		catchAsyncSignal(c)
		wg.Done()
	}()

	go func() {
		triggerSyncSignal()
		wg.Done()
	}()

	wg.Wait()
} 
```

构建并运行该例子后，先不断敲入“中断键(`ctrl+c`)”查看异步信号的处理动作；3秒后，同步信号被**除0计算**触发；最后我们用`kill`命令向该应用进程发送一个`SIGKILL`的不可捕获信号，我们来看看示例程序运行结果：

```go
$go build -o notify-signal go-program-notify-sync-and-async-signal.go
$./notify-signal 
^C收到异步信号: interrupt
^C收到异步信号: interrupt
恢复panic: runtime error: integer divide by zero
[1]    94498 killed     ./notify-signal 
```

如果多次调用`Notify`拦截某信号，但每次调用使用的channel是不同的，那么当应用进程收到异步信号时，Go运行时会给每个channel发送一份异步信号副本：

```go
// go-signal/go-program-notify-signal-twice.go
... ...
func main() {
	c1 := make(chan os.Signal, 1)
	c2 := make(chan os.Signal, 1)

	signal.Notify(c1, syscall.SIGINT, syscall.SIGTERM)
	signal.Notify(c2, syscall.SIGINT, syscall.SIGTERM)

	go func() {
		s := <-c1
		fmt.Println("c1: 收到异步信号", s)
	}()

	s := <-c2
	fmt.Println("c2: 收到异步信号", s)
	time.Sleep(5 * time.Second)
} 
```

运行该示例后，敲入“中断键(`ctrl+c`)”，我们看到如下结果：

```
$go run go-program-notify-signal-twice.go
^Cc2: 收到异步信号 interrupt
c1: 收到异步信号 interrupt 
```

我们看到虽然只触发一次异步信号，但由于有两个channel“订阅”对该信号的拦截事件，于是运行时在向`c1`发送一份信号的同时，又向`c2`发送了一份信号副本。

如果上述例子中`c1 == c2`，即在同一个channel上两次调用`Notify`函数(拦截同一异步信号)，那么当信号触发后，这个channel会不会受到两个信号呢？运行下面的示例，我们就能得到结果：

```go
// go-signal/go-program-notify-signal-twice-on-same-channel.go
... ...
func main() {
	var wg sync.WaitGroup
	c := make(chan os.Signal, 2)

	signal.Notify(c, syscall.SIGINT)
	signal.Notify(c, syscall.SIGINT)

	// Block until any signal is received.
	wg.Add(1)
	go func() {
		for {
			s := <-c
			fmt.Println("c: 收到异步信号", s)
		}
		wg.Done()
	}()
	wg.Wait()
} 
```

运行该示例后，不断敲入“中断键”，我们发现每次触发`SIGINT`信号，该程序都仅输出一行日志，即channel仅收到一个信号：

```go
$go run go-program-notify-signal-twice-on-same-channel.go
^Cc: 收到异步信号 interrupt
^Cc: 收到异步信号 interrupt
^Cc: 收到异步信号 interrupt
^Cc: 收到异步信号 interrupt
^\SIGQUIT: quit
... ... 
```

使用`Notify`函数后，用户层与运行时层的唯一联系就是channel。运行时收到异步信号后，会将信号写入channel。那么如果在用户层尚未来得及接收信号的时间段内，运行时连续多次收到触发信号，那么用户层是否可以收到全部信号呢？我们来看下面这个示例：

```go
// go-signal/go-program-notify-lost-signal.go
... ...
func main() {
	c := make(chan os.Signal, 1)
	signal.Notify(c, syscall.SIGINT)

	// 在这10s期间，我们多次触发SIGINT信号
	time.Sleep(10 * time.Second)

	for {
		select {
		case s := <-c:
			fmt.Println("c: 获取异步信号", s)
		default:
			fmt.Println("c: 没有信号, 退出")
			return
		}
	}
} 
```

运行该示例后，在10秒内连续敲入5次“中断键”，10秒后我们看到下面输出结果：

```go
$go run go-program-notify-block-signal.go
^C^C^C^C^Cc: 获取异步信号 interrupt
c: 没有信号, 退出 
```

我们看到用户层仅收到一个`SIGINT`信号，而其他四个都被“丢弃”了。我们将channel的缓冲区大小由1改为5，再来试一下：

```go
$go run go-program-notify-block-signal.go
^C^C^C^C^Cc: 获取异步信号 interrupt
c: 获取异步信号 interrupt
c: 获取异步信号 interrupt
c: 获取异步信号 interrupt
c: 获取异步信号 interrupt
c: 没有信号, 退出 
```

我们看到这回用户层收到了全部五个`SIGINT`信号。因此在使用`Notify`函数时，要根据业务场景的要求，适当选择channel缓冲区的大小。

## 3. 使用系统信号实现程序的优雅退出(gracefully exit)

所谓优雅退出，指的就是程序在退出前有机会等待尚未完成的事务处理、清理资源（比如关闭文件描述符、关闭socket）、保存必要中间状态、持久化内存数据（比如将内存中的数据刷新(flush)到文件中）等。

而与“优雅退出”对立的则是“强制退出”，也就是我们常说的使用`kill -9`，即`kill -s SIGKILL pid`。这个机制不会给目标进程任何时间空隙，而是直接将进程杀死，无论进程当前在做何种操作，这种操作常常导致“不一致”状态的出现。前面提过：`SIGKILL`是不可捕捉信号，进程无法有效针对该信号设置处理工作函数，因此我们不应该使用该信号作为优雅退出的触发机制。

Go常用来编写http服务，http服务如何优雅退出也是gopher们经常要考虑的问题，下面我们就用一个示例来说明一下如何结合系统信号的使用来实现http服务的优雅退出：

```go
// go-signal/go-program-exit-gracefully-with-notify.go 
... ...
func main() {
	var wg sync.WaitGroup

	http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprintf(w, "Hello, Signal!\n")
	})
	var srv = http.Server{
		Addr: "localhost:8080",
	}

	srv.RegisterOnShutdown(func() {
		// 在一个单独的goroutine中执行
		fmt.Println("clean resources on shutdown...")
		time.Sleep(2 * time.Second)
		fmt.Println("clean resources ok")
		wg.Done()
	})

	wg.Add(2)
	go func() {
		quit := make(chan os.Signal, 1)
		signal.Notify(quit, syscall.SIGINT,
			syscall.SIGTERM,
			syscall.SIGQUIT,
			syscall.SIGHUP)

		<-quit

		timeoutCtx, cf := context.WithTimeout(context.Background(), time.Second*5)
		defer cf()
		var done = make(chan struct{}, 1)
		go func() {
			if err := srv.Shutdown(timeoutCtx); err != nil {
				fmt.Printf("web server shutdown error: %v", err)
			} else {
				fmt.Println("web server shutdown ok")
			}
			done <- struct{}{}
			wg.Done()
		}()

		select {
		case <-timeoutCtx.Done():
			fmt.Println("web server shutdown timeout")
		case <-done:
		}
	}()

	err := srv.ListenAndServe()
	if err != nil {
		if err != http.ErrServerClosed {
			fmt.Printf("web server start failed: %v\n", err)
			return
		}
	}
	wg.Wait()
	fmt.Println("program exit ok")
} 
```

这是一个实现http服务优雅退出的典型方案：

- 首先，我们通过`Notify`捕获`SIGINT`、`SIGTERM`、`SIGQUIT`和`SIGHUP`这四个系统信号，这样当这四个信号中的任何一个触发时，我们的http服务都有机会在退出前做一些清理工作；
- 我们使用`http`包提供的`Shutdown`来实现http服务内部的退出清理工作：包括立即关闭所有listener、关闭所有空闲的连接、等待处于活动状态的连接处理完毕(变成空闲连接)等；
- `http.Server`还提供了`RegisterOnShutdown`方法以允许开发者注册`shutdown`时的回调函数。这是个在服务关闭前清理其他资源、做收尾工作的**好场所**；注册的函数将在一个单独的goroutine中执行，但`Shutdown`不会等待这些回调函数执行完毕。示例中我们使用一个`time.Sleep`来模拟清理函数带来的延时。

我们运行一下上面示例。启动示例后，敲入“中断键(`ctrl+c`”)开启http服务的优雅退出过程：

```go
$go run go-program-exit-gracefully-with-notify.go
^\web server shutdown ok
clean resources on shutdown...
clean resources ok
program exit ok 
```

## 4. 小结

本节要点：

- 了解系统信号的工作原理以及应用收到信号后的三种处理方式；
- 掌握Go对系统信号的封装原理：同步信号由Go运行时转换为运行时错误(panic)，异步信号通过Channel发送给用户层；
- 了解`Notify`的函数行为和使用注意事项；
- 掌握利用信号实现程序优雅退出的典型方案。