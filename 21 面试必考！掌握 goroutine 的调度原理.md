21 面试必考！掌握 goroutine 的调度原理

## 面试必考！掌握 goroutine 的调度原理

我们知道**并发**是一种能力，它让你的程序可以由若干个代码片段**组合**而成，并且每个片段都是独立运行的。Go 语言原生支持这种并发能力，而 **[goroutine](https://tip.golang.org/ref/spec#Go_statements)** 恰是 Go 原生支持并发的具体实现。无论是 Go 自身运行时代码还是用户层 Go 代码都无一例外地运行在 goroutine 中。

**goroutine 是由 Go 运行时管理的用户层轻量级线程**。相较于操作系统线程，goroutine 的资源占用和使用代价都要小得多。我们可以创建几十个、上百个甚至成千上百万的 goroutine，Go 的运行时负责对 goroutine 进行管理。所谓的管理就是 **“调度”**。粗糙地说**调度**就是决定何时哪个 goroutine 将获得资源开始执行、哪个 goroutine 应该停止执行让出资源、哪个 goroutine 应该被唤醒恢复执行等。goroutine 的调度本是 Go 语言开发团队应该关注的事情，大多数 gopher 们无需关心。但笔者觉得了解 goroutine 的调度模型和原理，对于编写出高质量的 Go 代码是大有裨益的。因此，在这一节中，我将和大家一起来探究一下 goroutine 调度器的原理[^1]和演化历史。

## 1. goroutine 调度器

提到“调度”，我们首先想到的就是操作系统对进程、线程的调度。操作系统调度器会将系统中的多个线程按照一定算法调度到物理 CPU 上去运行。正如上一节我们提到的：传统的编程语言比如 C、C++等的并发实现多是基于线程模型的，即应用程序负责创建线程(一般通过 libpthread 等库函数调用实现)，操作系统负责调度线程。这种传统支持并发的方式有诸多不足：

**复杂**

- 创建容易退出难：做过 C/C++编程的读者都知道，利用 libpthread 库中提供的 API 创建一个线程时，虽然要传入的参数个数不少，但好歹还是可以接受的。不过一旦涉及到线程的退出，就要考虑新创建的线程是否要与主线程分离(detach)，还是需要主线程等待子线程终止(join)并获取其终止状态？又或是否需要在新线程中设置取消点(cancel point)以保证被主线程取消(cancel)的时候能顺利退出；
- 并发执行单元间的通信困难且易错：多个线程之间的通信虽然有多种机制可选，但用起来也是相当复杂；并且一旦涉及到共享内存，就会用到各种锁互斥机制，死锁便成为家常便饭；
- 线程栈大小的设定：开发人员需选择使用默认的，还是自定义设置。

**难于规模化(scale)**

- 线程的使用代价虽然已经比进程小了很多，但我们依然不能大量创建线程，因为除了每个线程占用的资源不小之外，操作系统调度切换线程的代价也不小；
- 对于很多网络服务程序，由于不能大量创建线程，只能选择在少量线程里做网络多路复用的方案，即：使用 epoll/kqueue/IoCompletionPort 这套机制，即便有像 [libevent](https://github.com/libevent/libevent) 和 [libev](http://software.schmorp.de/pkg/libev.html) 这样的第三方库帮忙，写起这样的程序也是很不易的，存在大量钩子回调，给开发人员带来不小的心智负担。

为此，Go 采用了**用户层轻量级线程**的概念来解决这些问题，Go 将之称为"**goroutine**"。goroutine 占用的资源非常小，上一节提到过：Go 将每个 goroutine 栈的大小默认设置为 2k 字节。goroutine 调度的切换也不用陷入（trap）操作系统内核层完成，代价很低。因此，一个 Go 程序中可以创建成千上万个并发的 goroutine。而将这些 goroutine 按照一定算法放到“*CPU*”上执行的程序就称为**goroutine调度器**（**goroutine scheduler**）。

一个 Go 程序对于操作系统来说只是一个**用户层程序**，操作系统眼中只有线程，它甚至不知道有一种叫**goroutine**的事物的存在。goroutine 的调度全要靠 Go 自己完成。实现 Go 程序内 goroutine 之间“公平”的竞争“CPU”资源，这个任务就落到了 Go 运行时（runtime）头上了。要知道在一个 Go 程序中，除了用户层代码，剩下的就是 go 运行时了。

于是 goroutine 的调度问题就演变为 go 运行时如何将程序内的众多 goroutine 按照一定算法调度到“CPU”资源上运行了。在操作系统层面，线程竞争的“CPU”资源是真实的物理 CPU，但在 Go 程序层面，各个 goroutine 要竞争的"CPU"资源是什么呢？Go 程序是用户层程序，它本身就是整体运行在一个或多个操作系统线程上的，因此 goroutine 们要竞争的所谓“CPU”资源就是操作系统线程。这样 goroutine 调度器的任务就明确了：将 goroutine 按照一定算法放到不同的操作系统线程中去执行。这种在语言层面自带调度器的，我们称之为**原生支持并发**。

## 2. Go 调度器模型与演化过程

### 1) G-M 模型

2012 年 3 月 28 日，[Go 1.0 正式发布](https://blog.golang.org/go-version-1-is-released)。在这个版本中，Go 开发团队实现了一个简单的 goroutine 调度器。在这个调度器中，每个 goroutine 对应于运行时中的一个抽象结构：`G(goroutine)`，而被视作“物理CPU”的操作系统线程则被抽象为另外一个结构：`M(machine)`。这个模型实现起来比较简单且能正常工作，但是却存在着诸多问题。前英特尔黑带级工程师、现谷歌工程师 [Dmitry Vyukov](https://github.com/dvyukov) 在其《[Scalable Go Scheduler Design](https://docs.google.com/document/d/1TTj4T2JO42uD5ID9e89oa0sLKhJYD0Y_kqxDv3I3XMw/edit#!)》一文中指出了**G-M 模型**的一个重要不足： 限制了 Go 并发程序的伸缩性，尤其是对那些有高吞吐或并行计算需求的服务程序。问题主要体现在如下几个方面：

- 单一全局互斥锁`(Sched.Lock)`和集中状态存储的存在导致所有 goroutine 相关操作，比如：创建、重新调度等都要上锁；
- goroutine 传递问题：M 经常在 M 之间传递"可运行"的 goroutine，这导致调度延迟增大以及额外的性能损耗；
- 每个 M 都做内存缓存，导致内存占用过高，数据局部性较差；
- 由于系统调用(syscall)而形成的频繁的工作线程阻塞和解除阻塞，导致额外的性能损耗。

### 2) G-P-M 模型

于是 Dmitry Vyukov 亲自操刀改进了 Go 调度器，在[Go 1.1](https://golang.org/doc/go1.1)中实现了**G-P-M 调度模型**和[work stealing算法](http://supertech.csail.mit.edu/papers/steal.pdf)，这个模型一直沿用至今：

![21 面试必考！掌握 goroutine 的调度原理](https://img-hello-world.oss-cn-beijing.aliyuncs.com/d35ff10a95bb842965dfa7141bbfac57.png)

图6-1-1：goroutine的G-P-M调度模型

有名人曾说过：**“计算机科学领域的任何问题都可以通过增加一个间接的中间层来解决”**，Dmitry Vyukov的**G-P-M**模型恰是这一理论的践行者。Dmitry Vyukov 通过向 G-M 模型中增加了一个 P，使得 Go 调度器具有很好的伸缩性。

`P`是一个`“逻辑Proccessor”`，每个 G 要想真正运行起来，首先需要被分配一个 P，即进入到 P 的本地运行队列(local runq)中，这里暂忽略全局运行队列(global runq)那个环节。对于 G 来说，P 就是运行它的“CPU”，可以说：**在 G 的眼里只有 P**。但从 Go 调度器的视角来看，真正的“CPU”是 M，只有将 P 和 M 绑定才能让 P 的 runq 中的 G 得以真实运行起来。这样的 P 与 M 的关系，就好比 Linux 操作系统调度层面用户线程(user thread)与内核线程(kernel thread)的对应关系那样(N x M)。

### 3) 抢占式调度

G-P-M 模型的实现算是`Go调度器`的一大进步了，但调度器仍然有一个头疼的问题，那就是不支持抢占式调度，这导致一旦某个 G 中出现死循环的代码逻辑，那么 G 将永久占用分配给它的 P 和 M，而位于同一个 P 中的其他 G 将得不到调度，出现“**饿死**”的情况。更为严重的是，当只有一个 P 时(GOMAXPROCS=1)时，整个 Go 程序中的其他 G 都将“饿死”。于是 Dmitry Vyukov 又提出了《[Go Preemptive Scheduler Design](https://docs.google.com/document/d/1ETuA2IOmnaQ4j81AtTGT40Y4_Jr6_IDASEKg0t0dBR8/edit#!)》并在[Go 1.2](https://blog.golang.org/go12)中实现了“抢占式”调度。

这个抢占式调度的原理则是在每个函数或方法的入口，加上一段额外的代码，让运行时有机会检查是否需要执行抢占调度。这种解决方案只能说局部解决了“饿死”问题。对于没有函数调用而是纯算法循环计算的 G，Go 调度器依然无法抢占。

### 4) NUMA 调度模型

从 Go 1.2 以后，Go 将重点放在了对 GC 的低延迟的优化上了，对调度器的优化和改进似乎不那么热心了，只是伴随着 GC 的改进而作了些小的改动。Dmitry Vyukov 在 2014 年 9 月提出了一个新的设计草案文档：《[NUMA‐aware scheduler for Go](https://docs.google.com/document/u/0/d/1d3iI2QWURgDIsSR6G2275vMeQ_X7w-qxM2Vp7iGwwuM/pub)》，作为对未来 Go 调度器演进方向的一个提议，不过至今似乎这个提议也没有列入开发计划。

### 5) 其他优化

Go 运行时已经实现了[netpoller](http://morsmachine.dk/netpoller)，这使得即便 G 发起网络 I/O 操作也不会导致 M 被阻塞（仅阻塞 G），从而不会导致大量线程(M)被创建出来。但是对于常规文件的 I/O 操作一旦阻塞，那么线程(M)将进入挂起状态，等待 I/O 返回后被唤醒。这种情况下 P 将与挂起的 M 分离，再选择一个处于空闲状态(idle)的 M。如果此时没有空闲的 M，则会新创建一个 M(线程)，这就是为何大量 I/O 操作会导致大量线程被创建的原因。

Go 开发团队的[Ian Lance Taylor](https://github.com/ianlancetaylor)在[Go 1.9](https://golang.org/doc/go1.9)中增加了一个[针对文件 I/O 的 Poller(https://groups.google.com/forum/#!topic/golang-dev/tT8SoKfHty0)的功能，这个功能可以像 netpoller 那样，在 G 操作那些支持监听(pollable)的文件描述符时，仅会阻塞 G，而不会阻塞 M。不过该功能依然不能对常规文件有效，常规文件是不支持监听的(pollable)。但对于 Go 调度器而言，这也算是一个不小的进步了。

## 3. 对 Go 调度器原理的进一步理解

### 1) G、P、M

关于 G、P、M 的定义，我们可以参见`$GOROOT/src/runtime/runtime2.go`这个源文件。G、P、M 这三个结构体定义都是大块儿头，每个结构体定义都包含十几个甚至二、三十个字段。像调度器这样的核心代码向来很复杂，考虑的因素也非常多，代码“耦合”成一坨。不过从复杂的代码中，我们依然可以看出来 G、P、M 的各自大致用途，这里简要说明一下：

- G: 代表 goroutine，存储了 goroutine 的执行栈信息、goroutine 状态以及 goroutine 的任务函数等；另外 G 对象是可以重用的。
- P: 代表逻辑 processor，P 的数量决定了系统内最大可并行的 G 的数量（前提：系统的物理 CPU 核数>=P 的数量）；P 的最大作用还是其拥有的各种 G 对象队列、链表、一些缓存和状态。
- M: M 代表着真正的执行计算资源。在绑定有效的 P 后，进入一个调度循环；而调度循环的机制大致是从各种队列、P 的本地运行队列中获取 G，切换到 G 的执行栈上并执行 G 的函数，调用 goexit 做清理工作并回到 M。如此反复。M 并不保留 G 状态，这是 G 可以跨 M 调度的基础。

下面是 G、P、M 定义的代码片段(注意：本文使用的是 Go 1.12.7 版本，随着 Go 演化，结构体中的字段定义可能会有不同)：

```go
//src/runtime/runtime2.go
type g struct {
        stack      stack   // offset known to runtime/cgo
        sched     gobuf
        goid        int64
        gopc       uintptr // pc of go statement that created this goroutine
        startpc    uintptr // pc of goroutine function
        ... ...
}

type p struct {
    lock mutex

    id          int32
    status      uint32 // one of pidle/prunning/...
  
    mcache      *mcache
    racectx     uintptr

    // Queue of runnable goroutines. Accessed without lock.
    runqhead uint32
    runqtail uint32
    runq     [256]guintptr

    runnext guintptr

    // Available G's (status == Gdead)
    gfree    *g
    gfreecnt int32

  ... ...
}

type m struct {
    g0      *g     // goroutine with scheduling stack
    mstartfn      func()
    curg          *g       // current running goroutine
 .... ..
} 
```

### 2) G 被抢占调度

和操作系统按时间片调度线程不同，Go 并没有时间片的概念。如果某个 G 没有进行系统调用(syscall)、没有进行 I/O 操作、没有阻塞在一个 channel 操作上，那么**M 是如何让 G 停下来并调度下一个可运行的 G 的呢**？答案是：G 是被抢占调度的。

前面说过，除非极端的无限循环或死循环，否则只要 G 调用函数，Go 运行时就有了抢占 G 的机会。Go 程序启动时，运行时会去启动一个名为 sysmon 的 M(一般称为监控线程)，该 M 特殊之处在于其无需绑定 P 即可运行(以 g0 这个 G 的形式)，该 M 在整个 Go 程序的运行过程中至关重要：

```go
//$GOROOT/src/runtime/proc.go

// The main goroutine.
func main() {
     ... ...
    systemstack(func() {
        newm(sysmon, nil)
    })
    .... ...
}

// Always runs without a P, so write barriers are not allowed.
//
//go:nowritebarrierrec
func sysmon() {
    // If a heap span goes unused for 5 minutes after a garbage collection,
    // we hand it back to the operating system.
    scavengelimit := int64(5 * 60 * 1e9)
    ... ...

    if  .... {
        ... ...
        // retake P's blocked in syscalls
        // and preempt long running G's
        if retake(now) != 0 {
            idle = 0
        } else {
            idle++
        }
       ... ...
    }
} 
```

sysmon 每 20us~10ms 启动一次，sysmon 主要完成如下工作：

- 释放闲置超过 5 分钟的 span 物理内存；
- 如果超过 2 分钟没有垃圾回收，强制执行；
- 将长时间未处理的 netpoll 结果添加到任务队列；
- 向长时间运行的 G 任务发出抢占调度；
- 收回因 syscall 长时间阻塞的 P；

我们看到 sysmon 将“向长时间运行的 G 任务发出抢占调度”，这个事情由函数`retake`实施：

```go
// $GOROOT/src/runtime/proc.go

// forcePreemptNS is the time slice given to a G before it is
// preempted.
const forcePreemptNS = 10 * 1000 * 1000 // 10ms

func retake(now int64) uint32 {
          ... ...
           // Preempt G if it's running for too long.
            t := int64(_p_.schedtick)
            if int64(pd.schedtick) != t {
                pd.schedtick = uint32(t)
                pd.schedwhen = now
                continue
            }
            if pd.schedwhen+forcePreemptNS > now {
                continue
            }
            preemptone(_p_)
         ... ...
} 
```

可以看出，如果一个 G 任务运行 10ms，sysmon 就会认为其运行时间太久而发出抢占式调度的请求。一旦 G 的抢占标志位被设为 true，那么待这个 G 下一次调用函数或方法时，运行时便可以将 G 抢占并移出运行状态，放入 P 的本地运行队列(local runq)中，等待下一次被调度。

### 3) channel 阻塞或网络 I/O 情况下的调度

如果 G 被阻塞在某个 channel 操作或网络 I/O 操作上时，G 会被放置到某个等待(wait)队列中，而 M 会尝试运行 P 的下一个可运行的 G；如果此时 P 没有可运行的 G 供 M 运行，那么 M 将解绑 P，并进入挂起状态。当 I/O 操作完成或 channel 操作完成，在等待队列中的 G 会被唤醒，标记为可运行(runnable)，并被放入到某 P 的队列中，绑定一个 M 后继续执行。

### 4) 系统调用阻塞情况下的调度

如果 G 被阻塞在某个系统调用(system call)上，那么不光 G 会阻塞，执行该 G 的 M 也会解绑 P(实质是被 sysmon 抢走了)，与 G 一起进入挂起(sleep)状态。如果此时有空闲的 M，则 P 会与其绑定并继续执行其他 G；如果没有空闲的 M，但仍然有其他 G 要去执行，那么就会创建一个新 M(线程)。

当系统调用返回后，阻塞在该系统调用上的 G 会尝试获取一个可用的 P，如果没有可用的 P，那么 G 会被标记为 runnable，之前的那个挂起的 M 将再次进入挂起状态。

## 4. 调度器状态的查看方法

Go 提供了调度器当前状态的查看方法：使用 Go 运行时环境变量`GODEBUG`。比如下面例子：

```shell
$ GODEBUG=schedtrace=1000 godoc -http=:6060
SCHED 0ms: gomaxprocs=4 idleprocs=3 threads=3 spinningthreads=0 idlethreads=0 runqueue=0 [0 0 0 0]
SCHED 1001ms: gomaxprocs=4 idleprocs=0 threads=9 spinningthreads=0 idlethreads=3 runqueue=2 [8 14 5 2]
SCHED 2006ms: gomaxprocs=4 idleprocs=0 threads=25 spinningthreads=0 idlethreads=19 runqueue=12 [0 0 4 0]
SCHED 3006ms: gomaxprocs=4 idleprocs=0 threads=26 spinningthreads=0 idlethreads=8 runqueue=2 [0 1 1 0]
SCHED 4010ms: gomaxprocs=4 idleprocs=0 threads=26 spinningthreads=0 idlethreads=20 runqueue=12 [6 3 1 0]
SCHED 5010ms: gomaxprocs=4 idleprocs=0 threads=26 spinningthreads=1 idlethreads=20 runqueue=17 [0 0 0 0]
SCHED 6016ms: gomaxprocs=4 idleprocs=0 threads=26 spinningthreads=0 idlethreads=20 runqueue=1 [3 4 0 10]
... ... 
```

`GODEBUG`这个 Go 运行时环境变量很是强大，通过给其传入不同的`key1=value1,key2=value2...`组合，Go 的运行时会输出不同的调试信息，比如在这里我们给 GODEBUG 传入了`"schedtrace=1000"`，其含义就是每 1000ms 打印输出一次 goroutine 调度器的状态，每次一行。每一行各字段含义如下：

以上面例子中最后一行为例：

```shell
SCHED 6016ms: gomaxprocs=4 idleprocs=0 threads=26 spinningthreads=0 idlethreads=20 runqueue=1 [3 4 0 10] 
```

- **SCHED**：调试信息输出标志字符串，代表本行是 goroutine 调度器相关信息的输出；
- **6016ms**：即从程序启动到输出这行日志经过的时间；
- **gomaxprocs**：P 的数量；
- **idleprocs**：处于空闲状态(idle)的 P 的数量；通过 gomaxprocs 和 idleprocs 的差值，我们就可知道执行 go 代码的 P 的数量；
- **threads**：操作系统线程的数量，包含调度器使用的 M 数量，加上运行时自用的类似 sysmon 这样的线程的数量；
- **spinningthreads**：处于自旋(spin)状态的操作系统数量；
- **idlethread**：处于 idle 状态的操作系统线程的数量；
- **runqueue=1**： go 调度器全局运行队列中 G 的数量；
- `[3 4 0 10]`：分别为 4 个 P 的本地运行队列中的 G 的数量。

我们还可以输出每个 goroutine、M 和 P 的详细调度信息，但对于 Go 开发者来说，大多数情况这是不必要的：

```go
$ GODEBUG=schedtrace=1000,scheddetail=1 godoc -http=:6060

SCHED 0ms: gomaxprocs=4 idleprocs=3 threads=3 spinningthreads=0 idlethreads=0 runqueue=0 gcwaiting=0 nmidlelocked=0 stopwait=0 sysmonwait=0
  P0: status=1 schedtick=0 syscalltick=0 m=0 runqsize=0 gfreecnt=0
  P1: status=0 schedtick=0 syscalltick=0 m=-1 runqsize=0 gfreecnt=0
  P2: status=0 schedtick=0 syscalltick=0 m=-1 runqsize=0 gfreecnt=0
  P3: status=0 schedtick=0 syscalltick=0 m=-1 runqsize=0 gfreecnt=0
  M2: p=-1 curg=-1 mallocing=0 throwing=0 preemptoff= locks=1 dying=0 helpgc=0 spinning=false blocked=false lockedg=-1
  M1: p=-1 curg=17 mallocing=0 throwing=0 preemptoff= locks=0 dying=0 helpgc=0 spinning=false blocked=false lockedg=17
  M0: p=0 curg=1 mallocing=0 throwing=0 preemptoff= locks=1 dying=0 helpgc=0 spinning=false blocked=false lockedg=1
  G1: status=8() m=0 lockedm=0
  G17: status=3() m=1 lockedm=1

SCHED 1002ms: gomaxprocs=4 idleprocs=0 threads=13 spinningthreads=0 idlethreads=7 runqueue=6 gcwaiting=0 nmidlelocked=0 stopwait=0 sysmonwait=0

 P0: status=2 schedtick=2293 syscalltick=18928 m=-1 runqsize=12 gfreecnt=2
  P1: status=1 schedtick=2356 syscalltick=19060 m=11 runqsize=11 gfreecnt=0
  P2: status=2 schedtick=2482 syscalltick=18316 m=-1 runqsize=37 gfreecnt=1
  P3: status=2 schedtick=2816 syscalltick=18907 m=-1 runqsize=2 gfreecnt=4
  M12: p=-1 curg=-1 mallocing=0 throwing=0 preemptoff= locks=0 dying=0 helpgc=0 spinning=false blocked=true lockedg=-1
  M11: p=1 curg=6160 mallocing=0 throwing=0 preemptoff= locks=2 dying=0 helpgc=0 spinning=false blocked=false lockedg=-1
  M10: p=-1 curg=-1 mallocing=0 throwing=0 preemptoff= locks=0 dying=0 helpgc=0 spinning=false blocked=true lockedg=-1
 ... ...

SCHED 2002ms: gomaxprocs=4 idleprocs=0 threads=23 spinningthreads=0 idlethreads=5 runqueue=4 gcwaiting=0 nmidlelocked=0 stopwait=0 sysmonwait=0
  P0: status=0 schedtick=2972 syscalltick=29458 m=-1 runqsize=0 gfreecnt=6
  P1: status=2 schedtick=2964 syscalltick=33464 m=-1 runqsize=0 gfreecnt=39
  P2: status=1 schedtick=3415 syscalltick=33283 m=18 runqsize=0 gfreecnt=12
  P3: status=2 schedtick=3736 syscalltick=33701 m=-1 runqsize=1 gfreecnt=6
  M22: p=-1 curg=-1 mallocing=0 throwing=0 preemptoff= locks=0 dying=0 helpgc=0 spinning=false blocked=true lockedg=-1
  M21: p=-1 curg=-1 mallocing=0 throwing=0 preemptoff= locks=0 dying=0 helpgc=0 spinning=false blocked=true lockedg=-1
... ... 
```

关于 go 调度器调试信息输出的详细信息，可以参考 Dmitry Vyukov 的文章：《[Debugging performance issues in Go programs](https://software.intel.com/en-us/blogs/2014/05/10/debugging-performance-issues-in-go-programs)》。这也应该是每个 gopher 必读的经典文章。当然更详尽的代码可参考`$GOROOT/src/runtime/proc.go`中的`schedtrace`函数。

## 5. goroutine 调度实例简要分析

根据上面对 goroutine 调度器的理解，我们来看几个实例，并对实例进行一些简要分析。

### 1) 为何在存在死循环的情况下，多个 goroutine 依旧会被成功调度并轮流执行

我们先来看一个例子：

```go
// go-scheduler-model-case1.go
package main

import (
    "fmt"
    "time"
)

func deadloop() {
    for {
    }
}

func main() {
    go deadloop()
    for {
        time.Sleep(time.Second * 1)
        fmt.Println("I got scheduled!")
    }
} 
```

在上面实例中，我们启动了两个 goroutine，一个是 main goroutine，另外一个是运行 deadloop 函数(顾名思义，一个死循环)的 goroutine。main goroutine 为了展示方便，也用了一个“死循环”，并每隔一秒钟打印一条信息。下面是在笔者的 macbook pro 上运行这个例子（我的机器是四核八线程的，runtime 的 NumCPU 函数返回 8）：

```shell
$go run go-scheduler-model-case1.go
I got scheduled!
I got scheduled!
I got scheduled!
... ... 
```

从运行结果输出的日志来看，尽管有运行着死循环的 deadloop goroutine 的存在，main goroutine 仍然得到了调度。其根本原因在于机器是多核多线程的（这里指硬件线程，不是操作系统线程）。Go 从[1.5版本](http://tonybai.com/2015/07/10/some-changes-in-go-1-5/)之后将默认的 P 的数量由 1 改为 CPU 核的数量（实际上还乘以了每个核上硬线程数量）。这样上述例子在启动时创建了不止一个 P，我们用一幅图来直观诠释一下：

![21 面试必考！掌握 goroutine 的调度原理](https://img-hello-world.oss-cn-beijing.aliyuncs.com/1d6ad1b43dad1b56785361094963cddd.png)

图6-1-2：goroutine调度实例分析示意图-1

我们假设 deadloop goroutine 被调度到 P1 上，P1 在 M1(对应一个操作系统线程)上运行；而 main goroutine 被调度到 P2 上，P2 在 M2 上运行，M2 对应另外一个操作系统线程，而线程在操作系统调度层面被调度到物理的 CPU 核上运行，我们有多个 CPU 核，即便 deadloop 占满一个核，我们还可以在另外一个 CPU 核上运行 P2 上的 main goroutine，这也是 main goroutine 得到调度的原因。

### 2) 如何让 deadloop goroutine 以外的 goroutine 无法得到调度

如果我们非要 deadloop goroutine 以外的 goroutine 无法得到调度，我们该如何做呢？一种思路：让 Go 运行时不要启动那么多 P，让所有用户级的 goroutines 都在一个 P 上被调度。

下面是实现上述思路的三种办法：

- 在 main 函数的最开头处调用`runtime.GOMAXPROCS(1)`；
- 设置环境变量 export GOMAXPROCS=1 后再运行程序；
- 找一个单核单线程的机器（不过现在这样的机器太难找了，只能使用云服务器实现）

我们以第一种方法为例：

```go
// go-scheduler-model-case2.go
package main

import (
    "fmt"
    "runtime"
    "time"
)

func deadloop() {
    for {
    }
}

func main() {
    runtime.GOMAXPROCS(1)
    go deadloop()
    for {
        time.Sleep(time.Second * 1)
        fmt.Println("I got scheduled!")
    }
} 
```

运行这个程序后，你会发现 main goroutine 的`"I got scheduled"`再也无法输出了。这里的调度原理可以用下面图示说明：

![21 面试必考！掌握 goroutine 的调度原理](https://img-hello-world.oss-cn-beijing.aliyuncs.com/1fd91debde597c870148359210862b73.png)

图6-1-3：goroutine调度实例分析示意图-2

deadloop goroutine 在 P1 上被调度，由于 deadloop 内部逻辑没有给调度器任何抢占的机会，比如：进入`runtime.morestack_noctxt`。于是即便是 sysmon 这样的监控 goroutine，也仅仅是能给 deadloop goroutine 的抢占标志位设为 true 而已。由于 deadloop 内部没有任何进入调度器代码的机会，goroutine 重新调度始终无法发生。main goroutine 只能躺在 P1 的 local queue 中等待着。

注：Go 1.14版本加入了goroutine的抢占式调度，新的调度方式利用操作系统信号机制，因此在Go 1.14及后续版本中，上述例子将不适用。

### 3) 反转：如何在 GOMAXPROCS=1 的情况下让 main goroutine 得到调度呢？

我们做个反转：如何在 GOMAXPROCS=1 的情况下，让 main goroutine 也得到调度呢？有人说： “有函数调用，就有了进入调度器代码的机会”，我们来试验一下是否属实：

```go
// go-scheduler-model-case3.go
package main

import (
    "fmt"
    "runtime"
    "time"
)

func add(a, b int) int {
    return a + b
}

func deadloop() {
    for {
        add(3, 5)
    }
}

func main() {
    runtime.GOMAXPROCS(1)
    go deadloop()
    for {
        time.Sleep(time.Second * 1)
        fmt.Println("I got scheduled!")
    }
} 
```

我们在 deadloop goroutine 的 for 循环中加入了一个 add 函数调用。我们来运行一下这个程序，看是否能达成我们的目的：

```
$ go run go-scheduler-model-case3.go 
```

我们看到：`"I got scheduled!"`字样依旧没有出现在我们眼前！也就是说 main goroutine 没有得到调度！为什么呢？其实所谓的“有函数调用，就有了进入调度器代码的机会”，实际上是 go 编译器在函数的入口处插入了一个运行时的函数调用：`runtime.morestack_noctxt`。这个函数会检查是否需要扩容连续栈，并进入抢占调度的逻辑中。一旦所在 goroutine 被置为可被抢占的，那么抢占调度代码就会剥夺该 goroutine 的执行权，将其让给其他 goroutine。但是上面代码为什么没有实现这一点呢？我们需要在汇编层次看看 go 编译器生成的代码是什么样子的。

查看 Go 程序的汇编代码有许多种方法：

- 使用 objdump 工具：objdump -S go 二进制文件；
- 使用 gdb disassemble；
- 使用 go tool 工具生成汇编代码文件：go build -gcflags ‘-S’ xx.go > xx.s 2>&1
- 将 Go 代码编译成汇编代码：go tool compile -S xx.go > xx.s
- 使用 go tool 工具反编译 Go 程序：go tool objdump -S go-binary > xx.s

我们这里使用最后一种方法：利用`go tool objdump`反编译(并结合其他输出的汇编形式)：

```
$go build -o go-scheduler-model-case3 go-scheduler-model-case3.go
$go tool objdump -S go-scheduler-model-case3 > go-scheduler-model-case3.s 
```

打开 go-scheduler-model-case3.s，搜索 main.add，我们居然找不到这个函数的汇编代码，而 main.deadloop 的定义如下：

```
TEXT main.deadloop(SB) go-scheduler-model-case3.go
        for {
  0x1093a10             ebfe                    JMP main.deadloop(SB)

  0x1093a12             cc                      INT $0x3
  0x1093a13             cc                      INT $0x3
  0x1093a14             cc                      INT $0x3
  0x1093a15             cc                      INT $0x3
   ... ...
  0x1093a1f             cc                      INT $0x3 
```

我们看到 deadloop 中对 add 函数的调用并未出现。这显然是 go 编译器在生成代码时执行了优化的结果，因为 add 的调用对 deadloop 的行为结果没有任何影响。

我们关闭优化再来试试：

```
$go build -gcflags '-N -l' -o go-scheduler-model-case3-unoptimized go-scheduler-model-case3.go
$go tool objdump -S go-scheduler-model-case3-unoptimized > go-scheduler-model-case3-unoptimized.s 
```

我们打开文件`go-scheduler-model-case3-unoptimized.s`，查找 main.add，这回我们找到了它：

```
TEXT main.add(SB) go-scheduler-model-case3.go
func add(a, b int) int {
  0x1093a10             48c744241800000000      MOVQ $0x0, 0x18(SP)
        return a + b
  0x1093a19             488b442408              MOVQ 0x8(SP), AX
  0x1093a1e             4803442410              ADDQ 0x10(SP), AX
  0x1093a23             4889442418              MOVQ AX, 0x18(SP)
  0x1093a28             c3                      RET

  0x1093a29             cc                      INT $0x3
... ...
  0x1093a2f             cc                      INT $0x3 
```

deadloop 函数中也有了对 add 的显式调用：

```
TEXT main.deadloop(SB) go-scheduler-model-case3.go
  ... ...
  0x1093a51             48c7042403000000        MOVQ $0x3, 0(SP)
  0x1093a59             48c744240805000000      MOVQ $0x5, 0x8(SP)
  0x1093a62             e8a9ffffff              CALL main.add(SB)
        for {
  0x1093a67             eb00                    JMP 0x1093a69
  0x1093a69             ebe4                    JMP 0x1093a4f
... ... 
```

不过我们这个程序中的 main goroutine 依旧得不到调度，因为在 main.add 代码中，我们没有发现 morestack 函数的踪迹，也就是说即便调用了 add 函数，deadloop 也没有机会进入到 Go 运行时的 goroutine 调度逻辑中去。

为什么 Go 编译器没有在 main.add 函数中插入 morestack 的调用呢？那是因为 add 函数位于调用树的 leaf（叶子）位置，编译器可以确保其不再有新栈帧生成，不会导致栈分裂或超出现有栈边界，于是就不再插入 morestack。这样位于 morestack 中的调度器的抢占式检查也就无法得以执行。下面是`go build -gcflags '-S'`方式输出的 go-scheduler-model-case3.go 的汇编输出：

```shell
"".add STEXT nosplit size=19 args=0x18 locals=0x0
     TEXT    "".add(SB), NOSPLIT, $0-24
     FUNCDATA        $0, gclocals·54241e171da8af6ae173d69da0236748(SB)
     FUNCDATA        $1, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
     MOVQ    "".b+16(SP), AX
     MOVQ    "".a+8(SP), CX
     ADDQ    CX, AX
     MOVQ    AX, "".~r2+24(SP)
    RET 
```

我们看到了`nosplit`字样，这就说明 add 使用的栈是固定大小(24 个字节)，不会再分裂(split)或超出现有边界。关于在 for 循环中的叶子节点函数(leaf function)中是否应该插入 morestack 目前还有[一定争议](https://github.com/golang/go/issues/10958)，将来也许会对这样的情况做特殊处理。

既然明白了原理，我们就在 deadloop 和 add 函数之间再加入一个 dummy 函数，见下面代码：

```go
// go-scheduler-model-case4.go
package main

import (
    "fmt"
    "runtime"
    "time"
)

func add(a, b int) int {
    return a + b
}

func dummy() {
    add(3, 5)
}

func deadloop() {
    for {
        dummy()
    }
}

func main() {
    runtime.GOMAXPROCS(1)
    go deadloop()
    for {
        time.Sleep(time.Second * 1)
        fmt.Println("I got scheduled!")
    }
} 
```

执行该代码：

```shell
$go build -gcflags '-N -l' -o go-scheduler-model-case4 go-scheduler-model-case4.go 
$./go-scheduler-model-case4
I got scheduled!
I got scheduled!
I got scheduled! 
```

我们看到：main goroutine 果然得到了调度。我们再来看看 go 编译器为该程序生成的汇编代码：

```shell
$go build -gcflags '-N -l' -o go-scheduler-model-case4 go-scheduler-model-case4.go
$go tool objdump -S go-scheduler-model-case4 > go-scheduler-model-case4.s

TEXT main.add(SB) go-scheduler-model-case4.go
func add(a, b int) int {
  0x1093a10             48c744241800000000      MOVQ $0x0, 0x18(SP)
        return a + b
  0x1093a19             488b442408              MOVQ 0x8(SP), AX
  0x1093a1e             4803442410              ADDQ 0x10(SP), AX
  0x1093a23             4889442418              MOVQ AX, 0x18(SP)
  0x1093a28             c3                      RET

  0x1093a29             cc                      INT $0x3
  0x1093a2a             cc                      INT $0x3
... ...

TEXT main.dummy(SB) go-scheduler-model-case4.s
func dummy() {
  0x1093a30             65488b0c25a0080000      MOVQ GS:0x8a0, CX
  0x1093a39             483b6110                CMPQ 0x10(CX), SP
  0x1093a3d             762e                    JBE 0x1093a6d
  0x1093a3f             4883ec20                SUBQ $0x20, SP
  0x1093a43             48896c2418              MOVQ BP, 0x18(SP)
  0x1093a48             488d6c2418              LEAQ 0x18(SP), BP
        add(3, 5)
  0x1093a4d             48c7042403000000        MOVQ $0x3, 0(SP)
  0x1093a55             48c744240805000000      MOVQ $0x5, 0x8(SP)
  0x1093a5e             e8adffffff              CALL main.add(SB)
}
  0x1093a63             488b6c2418              MOVQ 0x18(SP), BP
  0x1093a68             4883c420                ADDQ $0x20, SP
  0x1093a6c             c3                      RET

  0x1093a6d             e86eacfbff              CALL runtime.morestack_noctxt(SB)
  0x1093a72             ebbc                    JMP main.dummy(SB)

  0x1093a74             cc                      INT $0x3
  0x1093a75             cc                      INT $0x3
  0x1093a76             cc                      INT $0x3
.... .... 
```

我们看到 main.add 函数依旧是叶子节点(leaf node)，没有插入 morestack 调用；但在新增的 dummy 函数中我们看到了`CALL runtime.morestack_noctxt(SB)`的身影。

### 4) 为何`runtime.morestack_noctxt(SB)`放到了 RET 后面？

在传统印象中，`runtime.morestack_noctxt`的调用应该是放在函数入口处的，但实际编译出来的汇编代码中(如上面函数 dummy 的汇编)，`runtime.morestack_noctxt(SB)`却放在了 RET 的后面。解释这个问题，我们最好来看一下另外一种形式的汇编输出(go build -gcflags '-S’方式输出的格式)：

```shell
"".dummy STEXT size=68 args=0x0 locals=0x20
        0x0000 00000 TEXT    "".dummy(SB), $32-0
        0x0000 00000 MOVQ    (TLS), CX
        0x0009 00009 CMPQ    SP, 16(CX)
        0x000d 00013 JLS     61
        0x000f 00015 SUBQ    $32, SP
        0x0013 00019 MOVQ    BP, 24(SP)
        0x0018 00024 LEAQ    24(SP), BP
        ... ...
        0x001d 00029 MOVQ    $3, (SP)
        0x0025 00037 MOVQ    $5, 8(SP)
        0x002e 00046 PCDATA  $0, $0
        0x002e 00046 CALL    "".add(SB)
        0x0033 00051 MOVQ    24(SP), BP
        0x0038 00056 ADDQ    $32, SP
        0x003c 00060 RET
        0x003d 00061 NOP
        0x003d 00061 PCDATA  $0, $-1
        0x003d 00061 CALL    runtime.morestack_noctxt(SB)
        0x0042 00066 JMP     0 
```

我们看到在函数入口处，compiler 插入三行汇编：

```shell
 0x0000 00000 MOVQ    (TLS), CX  // 将TLS的值(GS:0x8a0)放入CX寄存器
        0x0009 00009 CMPQ    SP, 16(CX)  //比较SP与CX+16的值
        0x000d 00013 JLS     61 // 如果SP > CX + 16，则jump到61这个位置，即runtime.morestack_noctxt(SB) 
```

这种形式输出的是标准 Plan9 的汇编语法，资料很少（比如 JLS 跳转指令的含义）。最后一行的含义是：如果跳转，则进入到`runtime.morestack_noctxt`，从`runtime.morestack_noctxt`返回后，再次跳转到开头执行(见最后一行的`JMP 0`)。

为什么要这么做呢？按照 go 开发团队的说法：这样做是为了更好的利用现代 CPU 的["静态分支预测(static branch prediction)"](https://github.com/golang/go/issues/10587)，可以提升执行性能。

## 6. 小结

本节要点：

- 了解 goroutine 调度器要解决的主要问题；
- goroutine 调度器的调度模型演进；
- goroutine 调度器当前``G-P-M`调度模型的运行原理；
- goroutine 调度器状态查看方法；
- goroutine 调度实例分析方法。