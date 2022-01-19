32 掌握 Go 代码性能剖析神器：pprof

## 掌握 Go 代码性能剖析神器：pprof

在上一节中，我们为代码建立起了性能基准，有了基准后，我们便可以知道代码是否遇到了性能瓶颈。对于那些确认遇到性能瓶颈的代码，我们需要知道瓶颈究竟在哪里。

Go 是“自带电池”（battery included）的语言，拥有着让其他主流语言羡慕的工具链，Go 同样也内置了对Go代码进行性能剖析的工具：**pprof**。pprof 源自[Google Perf Tools工具套件](https://github.com/gperftools/gperftools)，在 Go 发布早期就被集成到 Go 工具链中了，并且 Go 运行时原生支持输出满足 pprof 需要的性能采样数据。本节我们就一起来看一下如何通过 pprof 对 Go 代码中的性能瓶颈进行剖析和诊断。

## 1. pprof 的工作原理

使用 pprof 对程序进行性能剖析的工作一般分为两个阶段：**数据采集**和**数据剖析**，如下图所示：

![32 掌握 Go 代码性能剖析神器：pprof](https://img-hello-world.oss-cn-beijing.aliyuncs.com/ae42edb052fde53dcaf9c04f809a047e.png)

图8-5-1：pprof工作原理

### 1) 采样数据类型

在**数据采集阶段**，Go运行时会定期对剖析阶段所需的不同类型数据进行采样记录，当前主要支持的采样数据类型有如下几种：

- CPU 数据（对应上图中的`cpu.prof`）

CPU 类型采样数据是性能剖析中最常见的采样数据类型，它能帮助我们识别出代码关键路径上消耗 CPU 最多的函数。一旦启用 CPU 数据采样，Go 运行时会每隔一段短暂的时间（10ms）就中断一次(由 SIGPROF 信号引发)并记录当前所有 goroutine 的函数栈信息（存入`cpu.prof`）。

- 堆内存分配数据（对应上图中的`mem.prof`）。

堆内存分配采样数据和 CPU 采样数据一样，也是性能剖析中最常见的采样数据类型，它能帮助我们了解 Go 程序的当前和历史内存使用情况。堆内存分配的采样频率可配置，默认每 1000 次堆内存分配会做一次采样（存入`mem.prof`）。

- 锁竞争数据（对应上图中的`mutex.prof`）

锁竞争采样数据记录了当前 Go 程序中互斥锁争用而导致延迟的操作。如果你认为很大可能是因为互斥锁争用导致的 CPU 利用率不高，那么你可以为`go tool pprof`工具提供此类采样文件以供性能剖析阶段使用。该类型采样数据默认情况下是不启用的，请参见`runtime.SetMutexProfileFraction`或`go test -bench . xxx_test.go -mutexprofile mutex.out`启用它。

- 阻塞时间数据（对应上图中的`block.prof`）

该类型采样数据记录的是 goroutine 在某共享资源(一般是由同步原语保护)上的阻塞时间，包括从无缓冲 channel 收发数据、阻塞在一个已经被其他 goroutine 锁住的互斥锁、向一个满了的 channel 发送数据或从一个空的 channel 接收数据等。该类型采样数据默认情况下也是不启用的，请参见`runtime.SetBlockProfileRate`或`go test -bench . xxx_test.go -blockprofile block.out`启用它。

> 注意：采样不是免费的，因此**一次采样过程尽量仅采集一种类型的数据**，不要同时采样多种类型的数据，避免相互干扰采样结果。

### 2) 性能数据采集的方式

在上面的原理图中我们看到，Go目前主要支持两种性能数据采集方式：通过性能基准测试进行数据采集和独立程序的性能数据采集。

#### a) 通过性能基准测试进行数据采集

我们为应用中的关键函数/方法建立起性能基准测试之时，我们便可以通过性能基准测试的执行采集到整个测试执行过程中有关被测方法的各类性能数据。这种方式尤其适用于对应用中关键路径上关键函数/方法性能的剖析。

我们仅需为`go test`增加一些命令行选项即可在执行性能基准测试的同时进行性能数据采集，以**CPU采样数据类型**为例：

```shell
$go test -bench . xxx_test.go -cpuprofile=cpu.prof 
$ls 
cpu.prof xxx.test* xxx_test.go 
```

我们看到，一旦开启性能数据采集（比如传入**-cpuprofile**)，go test 的`-c`命令选项便会自动开启，`go test`命令执行后会自动编译出一个该测试对应的可执行文件（这里是`xxx.test`）。该二进制文件在性能数据剖析过程中可以提供剖析所需要的符号信息（比如：如果没有该二进制文件，`go tool pprof`的`disasm`命令将无法给出对应符号的汇编代码）。而`cpu.prof`就是存储CPU性能采样数据的结果文件，后续将**作为数据剖析过程的输入**。

对于其他类型的采样数据，我们也可以采用同样的方法开启采集并设置输出文件:

```shell
$go test -bench . xxx_test.go -memprofile=mem.prof 
$go test -bench . xxx_test.go -blockprofile=block.prof
$go test -bench . xxx_test.go -mutexprofile=mutex.prof 
```

#### b) 独立程序的性能数据采集

我们也可以通过标准库`runtime/pprof`和`runtime`包提供的低级API对独立运行的程序进行整体性能数据采集。下面是一个独立程序性能数据采集的例子：

```go
// pprof_standalone1.go 
package main

import (
	"flag"
	"fmt"
	"log"
	"os"
	"os/signal"
	"runtime"
	"runtime/pprof"
	"sync"
	"syscall"
	"time"
)

var cpuprofile = flag.String("cpuprofile", "", "write cpu profile to `file`")
var memprofile = flag.String("memprofile", "", "write memory profile to `file`")
var mutexprofile = flag.String("mutexprofile", "", "write mutex profile to `file`")
var blockprofile = flag.String("blockprofile", "", "write block profile to `file`")

func main() {
	flag.Parse()
	if *cpuprofile != "" {
		f, err := os.Create(*cpuprofile)
		if err != nil {
			log.Fatal("could not create CPU profile: ", err)
		}
		defer f.Close() // 该例子中暂忽略错误处理
		if err := pprof.StartCPUProfile(f); err != nil {
			log.Fatal("could not start CPU profile: ", err)
		}
		defer pprof.StopCPUProfile()
	}

	if *memprofile != "" {
		f, err := os.Create(*memprofile)
		if err != nil {
			log.Fatal("could not create memory profile: ", err)
		}
		defer f.Close()
		if err := pprof.WriteHeapProfile(f); err != nil {
			log.Fatal("could not write memory profile: ", err)
		}
	}

	if *mutexprofile != "" {
		runtime.SetMutexProfileFraction(1)
		defer runtime.SetMutexProfileFraction(0)
		f, err := os.Create(*mutexprofile)
		if err != nil {
			log.Fatal("could not create mutex profile: ", err)
		}
		defer f.Close()

		if mp := pprof.Lookup("mutex"); mp != nil {
			mp.WriteTo(f, 0)
		}
	}

	if *blockprofile != "" {
		runtime.SetBlockProfileRate(1)
		defer runtime.SetBlockProfileRate(0)
		f, err := os.Create(*blockprofile)
		if err != nil {
			log.Fatal("could not create block profile: ", err)
		}
		defer f.Close()

		if mp := pprof.Lookup("mutex"); mp != nil {
			mp.WriteTo(f, 0)
		}
	}

	var wg sync.WaitGroup
	c := make(chan os.Signal, 1)
	signal.Notify(c, syscall.SIGINT, syscall.SIGTERM)
	wg.Add(1)
	go func() {
		for {
			select {
			case <-c:
				wg.Done()
				return
			default:
				s1 := "hello,"
				s2 := "gopher"
				s3 := "!"
				_ = s1 + s2 + s3
			}

			time.Sleep(10 * time.Millisecond)
		}
	}()
	wg.Wait()
	fmt.Println("program exit")
} 
```

我们可以通过指定命令行参数的方式选择要采集的性能数据类型：

```shell
$go run pprof_standalone1.go -help
Usage of /var/folders/cz/sbj5kg2d3m3c6j650z0qfm800000gn/T/go-build221652171/b001/exe/pprof_standalone1:
  -blockprofile file
    	write block profile to file
  -cpuprofile file
    	write cpu profile to file
  -memprofile file
    	write memory profile to file
  -mutexprofile file
    	write mutex profile to file 
```

以CPU类型性能数据为例，我们执行下面命令：

```shell
$go run pprof_standalone1.go -cpuprofile cpu.prof
^Cprogram exit


$ls -l cpu.prof
-rw-r--r--  1 tonybai  staff  734  5 19 13:02 cpu.prof 
```

程序退出后，我们在当前目录下看到采集后的 CPU 类型性能数据结果文件`cpu.prof`，该文件将被提供给`go tool pprof`工具作后续剖析。

从上述示例我们看到，这种整体程序的性能数据采集方式对业务代码侵入较多，还要自己编写一些采集逻辑：定义 flag 变量、创建输出文件、关闭输出文件等。并且每次采集都要停止程序才能获取结果（当然我们也可以重新定义更复杂的控制采集时间窗口的逻辑，实现不停止程序也能获取采集数据结果）。

Go 在`net/http/pprof`包中还提供了另外一种更为高级的针对独立程序性能数据采集的方式，这种方式尤其适合那些为内置了 http 服务的独立程序，`net/http/pprof`包可以直接利用已有的 http 服务对外提供用于性能数据采集的服务端点（endpoint）。例如，如果一个已有的提供 http 服务的独立程序代码如下：

```go
// pprof_standalone2.go
package main

import (
	"context"
	"fmt"
	"log"
	"net/http"
	"os"
	"os/signal"
	"syscall"
)

func main() {
	http.Handle("/hello", http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		fmt.Println(*r)
		w.Write([]byte("hello"))
	}))
	s := http.Server{
		Addr: "localhost:8080",
	}
	c := make(chan os.Signal, 1)
	signal.Notify(c, syscall.SIGINT, syscall.SIGTERM)
	go func() {
		<-c
		s.Shutdown(context.Background())
	}()
	log.Println(s.ListenAndServe())
} 
```

如果我们要采集该http服务的性能数据，我们仅需在该独立程序的代码中像下面这样导入`net/http/pprof`包即可：

```go
// pprof_standalone2.go

import (
	... ...
	_ "net/http/pprof"
	... ...
) 
```

下面是`net/http/pprof`包的`init`函数，这就是空导入`net/http/pprof`的“副作用”：

```go
//$GOROOT/src/net/http/pprof/pprof.go

func init() {
        http.HandleFunc("/debug/pprof/", Index)
        http.HandleFunc("/debug/pprof/cmdline", Cmdline)
        http.HandleFunc("/debug/pprof/profile", Profile)
        http.HandleFunc("/debug/pprof/symbol", Symbol)
        http.HandleFunc("/debug/pprof/trace", Trace)
} 
```

我们看到该包的`init`函数向 http 包的默认请求“路由器(ServerMux)”：`DefaultServeMux)`注册了多个服务端点和对应的处理函数。而正是通过这些服务端点，我们就可以在该独立程序运行期间获取我们想要的各种类型的性能采集数据了。现在如果我们打开浏览器，访问`http://localhost:8080/debug/pprof/`，我们就可以看到下面的页面：

![32 掌握 Go 代码性能剖析神器：pprof](https://img-hello-world.oss-cn-beijing.aliyuncs.com/86836aec3fe677ba1c8632e27c68fb61.png)

图8-5-2：net/http/pprof提供的性能采集页面

这个页面里列出了多种类型的性能采集数据，点击其中任何一个即可完成该种类型性能数据的一次采集。`profile`是 CPU 类型数据的服务端点，我们点击该端点后，该服务默认会发起一次持续 30 秒的性能采集，得到的数据文件会通过浏览器自动下载到本地。如果想自定义采集时长，可以通过为服务端点传递时长参数（seconds）来实现，比如下面这个就是一个采样 60 秒的请求：

```shell
http://localhost:8080/debug/pprof/profile?seconds=60
```

如果独立程序的代码中没有使用http包的默认请求“路由器(ServerMux)”：`DefaultServeMux`，那我们就需要重新在新的“路由器”上为 pprof 包提供的性能数据采集方法注册服务端点，就像下面示例：

```go
// pprof_standalone3.go
... ...
func main() {
	mux := http.NewServeMux()
	mux.HandleFunc("/debug/pprof/", pprof.Index)
	mux.HandleFunc("/debug/pprof/profile", pprof.Profile)
	... ...
	mux.HandleFunc("/hello", http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		fmt.Println(*r)
		w.Write([]byte("hello"))
	}))
	s := http.Server{
		Addr:    "localhost:8080",
		Handler: mux,
	}
	c := make(chan os.Signal, 1)
	signal.Notify(c, syscall.SIGINT, syscall.SIGTERM)
	go func() {
		<-c
		s.Shutdown(context.Background())
	}()
	log.Println(s.ListenAndServe())
} 
```

如果是非http服务程序，则在导入包的同时，还需单独启动一个用于性能数据采集的goroutine，像下面这样：

```go
// pprof_standalone4.go

... ...
func main() {
	go func() {
		// 单独启动一个http server用于性能数据采集 
		fmt.Println(http.ListenAndServe("localhost:8080", nil))
	}()

	var wg sync.WaitGroup
	c := make(chan os.Signal, 1)
	signal.Notify(c, syscall.SIGINT, syscall.SIGTERM)
	wg.Add(1)
	go func() {
		for {
			select {
			case <-c:
				wg.Done()
				return
			default:
				s1 := "hello,"
				s2 := "gopher"
				s3 := "!"
				_ = s1 + s2 + s3
			}

			time.Sleep(100 * time.Millisecond)
		}
	}()
	wg.Wait()
	fmt.Println("program exit")
} 
```

通过上面几个示例，我们可以看出，导入`net/http/pprof`包进行独立程序性能数据采集的方式侵入性相对于第一种方式要小，代码也更为独立，并且可以在无需停止程序的情况下，通过预置好的各类性能数据采集服务端点，随时进行性能数据采集。

### 3) 性能数据的剖析

go 工具链通过`pprof`子命令提供了两种性能数据剖析的方法：**命令行交互式**和**Web图形化**。命令行交互式的剖析方法是最为常用，也是最基本的性能数据剖析方法；而基于 Web 图形化的剖析方法在剖析结果展示上相较于命令行交互式更为直观。

#### a) 命令行交互方式

我们可以通过下面三种方式执行`go tool pprof`以进入采用命令行交互式的性能数据剖析环节：

```shell
$go tool pprof xxx.test cpu.prof // 剖析通过性能基准测试采集的数据
$go tool pprof standalone_app cpu.prof // 剖析独立程序输出的性能采集数据
$go tool pprof http://localhost:8080/debug/pprof/profile // 通过net/http/pprof注册的性能采集数据服务端点获取数据并剖析 
```

下面我们以`pprof_standalone1.go`这个示例的性能采集数据为例，看一下在命令行交互式的剖析环节，我们的一些常用命令。我们首先生成 CPU 类型性能采集数据：

```shell
$go build -o pprof_standalone1 pprof_standalone1.go

$./pprof_standalone1 -cpuprofile pprof_standalone1_cpu.prof
^Cprogram exit 
```

通过`go tool pprof`命令进入到命令行交互模式中：

```shell
$ go tool pprof pprof_standalone1 pprof_standalone1_cpu.prof
File: pprof_standalone1
Type: cpu
Time: May 19, 2020 at 7:55pm (CST)
Duration: 16.14s, Total samples = 240ms ( 1.49%)
Entering interactive mode (type "help" for commands, "o" for options)
(pprof) 
```

从 pprof 子命令的输出中，我们看到：程序运行 16.14 秒，采样总时间为 240 毫秒，占总时间的 1.49%。

命令行交互方式下最常用的命令就是`topN`(N 为数字，如果不指定 N，默认 N 等于 10)：

```shell
(pprof) top
Showing nodes accounting for 240ms, 100% of 240ms total
Showing top 10 nodes out of 29
      flat  flat%   sum%        cum   cum%
      90ms 37.50% 37.50%       90ms 37.50%  runtime.nanotime1
      50ms 20.83% 58.33%       50ms 20.83%  runtime.pthread_cond_wait
      40ms 16.67% 75.00%       40ms 16.67%  runtime.usleep
      20ms  8.33% 83.33%       20ms  8.33%  runtime.asmcgocall
      20ms  8.33% 91.67%       20ms  8.33%  runtime.kevent
      10ms  4.17% 95.83%       10ms  4.17%  runtime.pthread_cond_signal
      10ms  4.17%   100%       10ms  4.17%  runtime.pthread_cond_timedwait_relative_np
         0     0%   100%       10ms  4.17%  main.main.func1
         0     0%   100%       30ms 12.50%  runtime.checkTimers
         0     0%   100%      130ms 54.17%  runtime.findrunnable
(pprof) 
```

`topN`命令的输出结果默认按`flat(flat%)`从大到小的顺序输出：

- `flat`列的值表示函数自身代码在数据采样过程的执行时长；
- `flat%`列的值表示函数自身代码在数据采样过程的执行时长占总采样执行时长的比例；
- `sum%`列的值是当前行`flat%`值与排在该值前面所有行的`flat%`值的累加和。以第三行的`sum%`值`75.00%`为例，该值由前三行`flat%`累加而得，即75.00% = 16.67% + 20.83% + 37.50%；
- `cum`列的值表示函数自身在数据采样过程中出现的时长，这个时长是其自身代码执行时长以及其等待其调用的函数返回所用时长的总和。越是接近函数调用栈底层的代码，其`cum`列的值越大；
- `cum%`列的值表示该函数`cum`值占总采样时长的百分比。比如：`runtime.findrunnable`函数的`cum`值为 130ms，总采样时长为 240ms，则其```cum%值为两者的比值百分化后的值(130.0/240，再百分化)。

命令行交互模式也支持按`cum`值从大到小排序输出采样结果：

```shell
(pprof) top -cum
Showing nodes accounting for 90ms, 37.50% of 240ms total
Showing top 10 nodes out of 29
      flat  flat%   sum%        cum   cum%
         0     0%     0%      140ms 58.33%  runtime.mcall
         0     0%     0%      140ms 58.33%  runtime.park_m
         0     0%     0%      140ms 58.33%  runtime.schedule
         0     0%     0%      130ms 54.17%  runtime.findrunnable
         0     0%     0%       90ms 37.50%  runtime.nanotime (inline)
      90ms 37.50% 37.50%       90ms 37.50%  runtime.nanotime1
         0     0% 37.50%       70ms 29.17%  runtime.mstart
         0     0% 37.50%       70ms 29.17%  runtime.mstart1
         0     0% 37.50%       70ms 29.17%  runtime.sysmon
         0     0% 37.50%       60ms 25.00%  runtime.semasleep
(pprof) 
```

在命令行交互模式下，我们可以通过`list`命令列出函数对应的源码，比如我们列出`main.main`函数的源码：

```shell
(pprof) list main.main
Total: 240ms
ROUTINE ======================== main.main.func1 in chapter8/sources/pprof_standalone1.go
         0       10ms (flat, cum)  4.17% of Total
         .          .     86:				s2 := "gopher"
         .          .     87:				s3 := "!"
         .          .     88:				_ = s1 + s2 + s3
         .          .     89:			}
         .          .     90:
         .       10ms     91:			time.Sleep(10 * time.Millisecond)
         .          .     92:		}
         .          .     93:	}()
         .          .     94:	wg.Wait()
         .          .     95:	fmt.Println("program exit")
         .          .     96:}
(pprof) 
```

我们看到：在展开源码的同时，pprof还列出了代码中对应行的消耗时长（基于采样数据）。我们可以选择耗时较长的函数，做进一步的向下展开，这个过程类似一个对代码进行向下钻取的过程，直到找到令我们满意的结果（某个导致性能瓶颈的函数中的某段代码）

```shell
(pprof) list time.Sleep
Total: 240ms
ROUTINE ======================== time.Sleep in go1.14/src/runtime/time.go
         0       10ms (flat, cum)  4.17% of Total
         .          .    192:		t = new(timer)
         .          .    193:		gp.timer = t
         .          .    194:	}
         .          .    195:	t.f = goroutineReady
         .          .    196:	t.arg = gp
         .       10ms    197:	t.nextwhen = nanotime() + ns
         .          .    198:	gopark(resetForSleep, unsafe.Pointer(t), waitReasonSleep, traceEvGoSleep, 1)
         .          .    199:}
         .          .    200:
(pprof) 
```

在命令行交互模式下，我们还可以生成 CPU 采样数据的函数调用图，图可以导出为多种格式：pdf、png、jpg、gif、svg 等。不过要做到这一点，前提是本地要安装图片生成依赖的插件：**graphviz**。

我们导出一幅 png 格式的图片：

```shell
(pprof) png
Generating report in profile001.png 
```

`png`命令在当前目录下生成了一幅名为`profile001.png`的图片文件：

![32 掌握 Go 代码性能剖析神器：pprof](https://img-hello-world.oss-cn-beijing.aliyuncs.com/1256ae7f6602c55772b42d3cbfa20b5c.png)

图8-5-3：pprof_standalone1采样数据生成的函数调用图

在上面图中，我们可以清晰地看到`cum%`较大的叶子节点（用黑色粗体标出，叶子节点的`cum%`值与`flat%`值相等），它们就是我们需要重点关注的优化点。

在命令行交互模式下，通过`web`命令还可以在输出 svg 格式图片的同时自动打开本地浏览器展示生成的 svg 图片。要实现这个功能也有一个前提，那就是本地的 svg 格式的默认打开应用应该设置为你的浏览器，否则生成的 svg 文件很可能以其他文本形式被其他应用（比如:vscode）打开。

#### b) Web图形化方式

对于喜好通过 GUI 方式剖析程序性能的开发者，`go tool pprof`同样提供了基于 Web 的图形化呈现采集的性能数据的方式。对于已经生成好的各类性能采集数据文件，我们可以通过下面命令行启动一个 Web 服务并自动打开本地浏览器进入图形化剖析页面：

```shell
$go tool pprof -http=:9090 pprof_standalone1_cpu.prof
Serving web UI on http://localhost:9090 
```

![32 掌握 Go 代码性能剖析神器：pprof](https://img-hello-world.oss-cn-beijing.aliyuncs.com/1418cb4ff2ac6d69e2855e5f507c52c4.png)

图8-5-4：go tool pprof自动打开本地浏览器进入图形化剖析页面

我们看到图形化剖析页面的默认视图(view)是“Graph”，即函数调用图。在左上角的“VIEW”下拉菜单中，我们还可以看到“Top”、“Flame Graph”、“Source”等几个菜单项：

- Top 视图：等价于命令行交互模式下的 topN 命令输出。

![32 掌握 Go 代码性能剖析神器：pprof](https://img-hello-world.oss-cn-beijing.aliyuncs.com/eb70050097ebdadceb67c10abbc9f57f.png)

图8-5-5：图形化剖析页面的Top视图

- Source 视图：等价于命令行交互模式下的 list 命令输出，只是这里将所有采样到的函数相关源码在一个页面全部列出了。

![32 掌握 Go 代码性能剖析神器：pprof](https://img-hello-world.oss-cn-beijing.aliyuncs.com/7e0ee7e712de0dfb900e68104900ce96.png)

图8-5-6：图形化剖析页面的Source视图

- Flame Graph视图：即[火焰图](http://www.brendangregg.com/flamegraphs.html)，该类型视图由性能架构师 Brendan Gregg 发明，并在近几年被广大开发人员接受。Go 1.10 版本在 go 工具链中添加了对火焰图的支持。通过火焰图，我们可以快速准确地识别出执行最频繁的代码路径，因此它多用于对 CPU 类型采集数据的辅助剖析(其它类型的性能采样数据也有对应的火焰图，比如：内存分配)。

![32 掌握 Go 代码性能剖析神器：pprof](https://img-hello-world.oss-cn-beijing.aliyuncs.com/4e3e81a1d45ff92a53d7aefcc0d96117.png)

图8-5-7：图形化剖析页面的Flame Graph视图

我们看到：`go tool pprof`在浏览器中呈现出的火焰图与标准火焰图有些差异：它是倒置的，即调用栈最顶端的函数在最下方。在这样一幅倒置火焰图中，y轴表示函数调用栈，每一层都是一个函数。调用栈越深，火焰越高。倒置火焰图每个函数调用栈的最下方就是正在执行的函数，上方都是它的父函数。

火焰图的 x 轴表示抽样数量，如果一个函数在 x 轴占据的宽度越宽，就表示它被抽样到的次数越多，即执行的时间越长。倒置火焰图就是看最下面的哪个函数占据的宽度最大，这样的函数可能存在性能问题。

鼠标悬浮在火焰图上的任意一个函数上时，图上方会显示该函数的性能采样详细信息。在火焰图上任意点击某个函数栈上的函数，火焰图都会水平局部放大，该函数会占据所在层全部宽度，显示更为详细的信息。再点击 root 层或 REFINE 下拉菜单中的 Reset 可恢复火焰图原来的样子。

对于通过`net/http/pprof`暴露性能数据采样端点的独立程序，我们同样可以采用基于Web的图形化页面进行性能剖析，以`pprof_standalone4.go`的剖析为例：

```shell
// 启动pprof_standalone4.go
$go run pprof_standalone4.go

// 启动Web图形化剖析
$go tool pprof -http=:9090 http://localhost:8080/debug/pprof/profile
Fetching profile over HTTP from http://localhost:8080/debug/pprof/profile
Saved profile in /Users/tonybai/pprof/pprof.samples.cpu.001.pb.gz
Serving web UI on http://localhost:9090 
```

执行`go tool pprof`时，pprof 会对`pprof_standalone4.go`进行默认 30 秒的 CPU 类型性能数据采样，然后将采集的数据下载到本地，存为`pprof.samples.cpu.001.pb.gz`，之后`go tool pprof`加载`pprof.samples.cpu.001.pb.gz`并自动启动浏览器进入性能剖析默认页面(函数调用图)：

![32 掌握 Go 代码性能剖析神器：pprof](https://img-hello-world.oss-cn-beijing.aliyuncs.com/7ea22c27a72626d2f7db4da6f0959a60.png)

图8-5-8：针对采用net/http/pprof的独立程序的Web图形化剖析页面

剩下的操作和之前描述的完全一致，这里就不赘述了。

## 2. 使用 pprof 进行性能剖析的实例

前面我们了解了`go tool pprof`的工作原理、性能数据类别、采样方式以及剖析方式等，下面我们用一个实例来整体说明一下利用 pprof 进行性能剖析的过程。该示例改编自 Brad Fitzpatrick 在[YAPC Asia 2015](http://yapcasia.org/)上的一次名为“Go Debugging, Profiling, and Optimization”的技术分享。

### 1) 待优化程序(step0)

待优化程序是一个简单的 http 服务，当通过浏览器访问其`/hi`服务端点时，页面上会显示下面内容：

![32 掌握 Go 代码性能剖析神器：pprof](https://img-hello-world.oss-cn-beijing.aliyuncs.com/c2ab902aaf48d668caf2dd8a17e2b933.png)

图8-5-9：演示程序呈现的页面

我们看到页面上有一个计数器，显示你是网站的第几个访客。同时该页面还支持通过 color 参数进行标题颜色定制，比如使用浏览器访问下面地址后，页面显示的"Welcome"标题将变成红色。

```shell
http://localhost:8080/hi?color=red 
```

该待优化程序的源码如下：

```go
//go-pprof-optimization-demo/step0/demo.go
package main

import (
	"fmt"
	"log"
	"net/http"
	"regexp"
	"sync/atomic"
)

var visitors int64 // must be accessed atomically

func handleHi(w http.ResponseWriter, r *http.Request) {
	if match, _ := regexp.MatchString(`^\w*$`, r.FormValue("color")); !match {
		http.Error(w, "Optional color is invalid", http.StatusBadRequest)
		return
	}
	visitNum := atomic.AddInt64(&visitors, 1)
	w.Header().Set("Content-Type", "text/html; charset=utf-8")
	w.Write([]byte("<h1 style='color: " + r.FormValue("color") +
		"'>Welcome!</h1>You are visitor number " + fmt.Sprint(visitNum) + "!"))
}

func main() {
	log.Printf("Starting on port 8080")
	http.HandleFunc("/hi", handleHi)
	log.Fatal(http.ListenAndServe("127.0.0.1:8080", nil))
} 
```

这里我们的实验环境为go 1.14 + macOS 10.14.6。

### 2) CPU类性能数据采样及数据剖析(step1)

前面提到`go tool pprof`支持多种类型的性能数据采集和剖析，我们大多数情况都会先从CPU类性能数据的剖析开始。这里我们通过为示例程序建立性能基准测试的方式采集CPU类性能数据。

```go
//go-pprof-optimization-demo/step1/demo_test.go
... ...
func BenchmarkHi(b *testing.B) {
	req, err := http.ReadRequest(bufio.NewReader(strings.NewReader("GET /hi HTTP/1.0\r\n\r\n")))
	if err != nil {
		b.Fatal(err)
	}
	rw := httptest.NewRecorder()
	b.ResetTimer()

	for i := 0; i < b.N; i++ {
		handleHi(rw, req)
	}
}
... ... 
```

建立基准，取得基准测试数据：

```shell
$go test -v -run=^$ -bench=.
goos: darwin
goarch: amd64
pkg: chapter8/sources/go-pprof-optimization-demo/step1
BenchmarkHi
BenchmarkHi-8   	  365084	      3218 ns/op
PASS
ok  	chapter8/sources/go-pprof-optimization-demo/step1	2.069s 
```

接下来，我们利用基准测试采样 CPU 类型性能数据：

```shell
$go test -v -run=^$ -bench=^BenchmarkHi$ -benchtime=2s -cpuprofile=cpu.prof 
```

执行完上述命令后，step1 目录下会出现两个新文件`step1.test`和`cpu.prof`，我么将这两个文件作为`go tool pprof`的输入对性能数据进行剖析：

```go
$go tool pprof step1.test cpu.prof
File: step1.test
Type: cpu
Time: xx
Duration: 2.35s, Total samples = 2.31s (98.44%)
Entering interactive mode (type "help" for commands, "o" for options)
(pprof) top
Showing nodes accounting for 1470ms, 63.64% of 2310ms total
Dropped 43 nodes (cum <= 11.55ms)
Showing top 10 nodes out of 121
      flat  flat%   sum%        cum   cum%
     480ms 20.78% 20.78%      480ms 20.78%  runtime.memmove
     180ms  7.79% 28.57%      180ms  7.79%  runtime.madvise
     160ms  6.93% 35.50%      750ms 32.47%  runtime.mallocgc
     130ms  5.63% 41.13%      130ms  5.63%  runtime.memclrNoHeapPointers
     110ms  4.76% 45.89%      130ms  5.63%  runtime.heapBitsSetType
     110ms  4.76% 50.65%      110ms  4.76%  runtime.nextFreeFast (inline)
     100ms  4.33% 54.98%      280ms 12.12%  regexp.makeOnePass.func1
     100ms  4.33% 59.31%      440ms 19.05%  runtime.growslice
      50ms  2.16% 61.47%       50ms  2.16%  runtime.(*mspan).refillAllocCache
      50ms  2.16% 63.64%       50ms  2.16%  runtime.nanotime1
(pprof) top -cum
Showing nodes accounting for 0.18s, 7.79% of 2.31s total
Dropped 43 nodes (cum <= 0.01s)
Showing top 10 nodes out of 121
      flat  flat%   sum%        cum   cum%
         0     0%     0%      1.90s 82.25%  chapter8/sources/go-pprof-optimization-demo/step1.BenchmarkHi
         0     0%     0%      1.90s 82.25%  chapter8/sources/go-pprof-optimization-demo/step1.handleHi
         0     0%     0%      1.90s 82.25%  testing.(*B).launch
         0     0%     0%      1.90s 82.25%  testing.(*B).runN
         0     0%     0%      1.31s 56.71%  regexp.MatchString
         0     0%     0%      1.26s 54.55%  regexp.Compile (inline)
     0.01s  0.43%  0.43%      1.26s 54.55%  regexp.compile
     0.16s  6.93%  7.36%      0.75s 32.47%  runtime.mallocgc
     0.01s  0.43%  7.79%      0.49s 21.21%  regexp/syntax.Parse
         0     0%  7.79%      0.48s 20.78%  bytes.(*Buffer).Write
(pprof) 
```

通过`top -cum`，我们看到`handleHi`累积消耗CPU最多(用户层代码范畴)，通过`list`命令进一步展开`handleHi`函数：

```shell
(pprof) list handleHi
Total: 2.31s
ROUTINE ======================== chapter8/sources/go-pprof-optimization-demo/step1.handleHi in chapter8/sources/go-pprof-optimization-demo/step1/demo.go
         0      1.90s (flat, cum) 82.25% of Total
         .          .      9:)
         .          .     10:
         .          .     11:var visitors int64 // must be accessed atomically
         .          .     12:
         .          .     13:func handleHi(w http.ResponseWriter, r *http.Request) {
         .      1.31s     14:	if match, _ := regexp.MatchString(`^\w*$`, r.FormValue("color")); !match {
         .          .     15:		http.Error(w, "Optional color is invalid", http.StatusBadRequest)
         .          .     16:		return
         .          .     17:	}
         .          .     18:	visitNum := atomic.AddInt64(&visitors, 1)
         .       30ms     19:	w.Header().Set("Content-Type", "text/html; charset=utf-8")
         .      500ms     20:	w.Write([]byte("<h1 style='color: " + r.FormValue("color") +
         .       60ms     21:		"'>Welcome!</h1>You are visitor number " + fmt.Sprint(visitNum) + "!"))
         .          .     22:}
         .          .     23:
         .          .     24:func main() {
         .          .     25:	log.Printf("Starting on port 8080")
         .          .     26:	http.HandleFunc("/hi", handleHi)
(pprof) 
```

我们看到在`handleHi`中，`MatchString`函数调用耗时最长(1.31秒)。

### 3) 第一次优化(step2)

通过前面对 CPU 类性能数据的剖析，我们已经发现`MatchString`较为耗时。通过阅读代码发现，每次 http 服务接收请求后，都会采用正则表达式对请求中的`color`参数值做一次匹配校验。校验使用的是`regexp`包的`MatchString`函数，该函数每次执行都要重新编译传入的正则表达式，因此速度较慢。我们的优化手段：**让正则式仅编译一次**。下面是优化后的代码：

```go
//go-pprof-optimization-demo/step2/demo.go

... ...
var visitors int64 // must be accessed atomically

var rxOptionalID = regexp.MustCompile(`^\d*$`)

func handleHi(w http.ResponseWriter, r *http.Request) {
    if !rxOptionalID.MatchString(r.FormValue("color")) {
        http.Error(w, "Optional color is invalid", http.StatusBadRequest)
        return
    }

    visitNum := atomic.AddInt64(&visitors, 1)
    w.Header().Set("Content-Type", "text/html; charset=utf-8")
    w.Write([]byte("<h1 style='color: " + r.FormValue("color") +
        "'>Welcome!</h1>You are visitor number " + fmt.Sprint(visitNum) + "!"))
}
... ... 
```

在优化后的代码中，我们使用一个代表编译后正则表达式对象的`rxOptionalID`的`MatchString`方法替换掉了每次都需要重新编译正则表达式的`MatchString`函数调用。

重新运行一下性能基准测试：

```shell
$go test -v -run=^$ -bench=.
goos: darwin
goarch: amd64
pkg: chapter8/sources/go-pprof-optimization-demo/step2
BenchmarkHi
BenchmarkHi-8   	 2624650	       457 ns/op
PASS
ok  	chapter8/sources/go-pprof-optimization-demo/step2	1.734s 
```

我们看到相比于优化前的`3218 ns/op`，优化后的`457 ns/op`将`handleHi`的性能提高了 7 倍多。

### 4) 内存分配采样数据剖析

在对待优化程序进行完 CPU 类型性能数据剖析以及优化实施之后，我们再来采集一下另外一种最常用的性能采样数据：内存分配类型数据，探索一下在内存分配方面是否有可以优化的地方。Go 程序内存分配一旦过频过多，就会大幅增加 Go GC 的工作负荷，导致 GC 延迟增大，从而影响应用的整体性能。因此，优化内存分配行为一定程度上也是提升应用程序性能的手段。

在`go-pprof-optimization-demo/step2`目录下，我们为`demo_test.go`中的`BenchmarkHi`增加`ReportAllocs`方法调用，让其输出内存分配信息。然后，我们通过性能基准测试的执行获取内存分配采样数据：

```shell
$go test -v -run=^$ -bench=^BenchmarkHi$ -benchtime=2s -memprofile=mem.prof
goos: darwin
goarch: amd64
pkg: chapter8/sources/go-pprof-optimization-demo/step2
BenchmarkHi
BenchmarkHi-8   	 5243474	       455 ns/op	     364 B/op	       5 allocs/op
PASS
ok  	chapter8/sources/go-pprof-optimization-demo/step2	3.052s 
```

接下来，我们就使用 pprof 工具剖析输出的内存分配采用数据(`mem.prof`)：

```shell
$go tool pprof step2.test mem.prof
File: step2.test
Type: alloc_space
Entering interactive mode (type "help" for commands, "o" for options)
(pprof) 
```

在`go tool pprof`的输出中有一行为`Type: alloc_space`。这行的含义是当前 pprof 将呈现的是程序运行期间的所有内存分配的采样数据（即使该分配的内存在最后一次采样时已经被释放）; 我们还可以让 pprof 将`Type`切换为`inuse_space`，这个类型表示的是内存数据采样结束时依然在用的内存。

我们可以在启动pprof工具时指定所使用的内存数据呈现类型：

```shell
$go tool pprof --alloc_space step2.test mem.prof // 遗留方式
$go tool pprof -sample_index=alloc_space step2.test mem.prof //最新方式 
```

亦可在进入 pprof 交互模式后，通过`sample_index`命令实现切换：

```shell
(pprof) sample_index = inuse_space 
```

我们现在以`alloc_space`类型进入 pprof 命令交互界面并执行 top 命令：

```shell
$go tool pprof -sample_index=alloc_space step2.test mem.prof 
File: step2.test
Type: alloc_space
Entering interactive mode (type "help" for commands, "o" for options)
(pprof) top
Showing nodes accounting for 2.05GB, 100% of 2.05GB total
Showing top 10 nodes out of 11
      flat  flat%   sum%        cum   cum%
    1.12GB 54.82% 54.82%     1.12GB 54.82%  bytes.makeSlice
    0.82GB 40.10% 94.92%     2.05GB   100%  chapter8/sources/go-pprof-optimization-demo/step2.handleHi
    0.09GB  4.53% 99.45%     0.09GB  4.53%  net/textproto.MIMEHeader.Set (inline)
    0.01GB  0.55%   100%     0.01GB  0.55%  fmt.Sprint
         0     0%   100%     1.12GB 54.82%  bytes.(*Buffer).Write
         0     0%   100%     1.12GB 54.82%  bytes.(*Buffer).grow
         0     0%   100%     2.05GB   100%  chapter8/sources/go-pprof-optimization-demo/step2.BenchmarkHi
         0     0%   100%     0.09GB  4.53%  net/http.Header.Set (inline)
         0     0%   100%     1.12GB 54.82%  net/http/httptest.(*ResponseRecorder).Write
         0     0%   100%     2.05GB   100%  testing.(*B).launch
(pprof) top -cum
Showing nodes accounting for 2084.53MB, 99.45% of 2096.03MB total
Showing top 10 nodes out of 11
      flat  flat%   sum%        cum   cum%
         0     0%     0%  2096.03MB   100%  chapter8/sources/go-pprof-optimization-demo/step2.BenchmarkHi
  840.55MB 40.10% 40.10%  2096.03MB   100%  chapter8/sources/go-pprof-optimization-demo/step2.handleHi
         0     0% 40.10%  2096.03MB   100%  testing.(*B).launch
         0     0% 40.10%  2096.03MB   100%  testing.(*B).runN
         0     0% 40.10%  1148.98MB 54.82%  bytes.(*Buffer).Write
         0     0% 40.10%  1148.98MB 54.82%  bytes.(*Buffer).grow
 1148.98MB 54.82% 94.92%  1148.98MB 54.82%  bytes.makeSlice
         0     0% 94.92%  1148.98MB 54.82%  net/http/httptest.(*ResponseRecorder).Write
         0     0% 94.92%       95MB  4.53%  net/http.Header.Set (inline)
      95MB  4.53% 99.45%       95MB  4.53%  net/textproto.MIMEHeader.Set (inline)
(pprof) 
```

我们看到`handleHi`分配了较多内存。我们通过`list`命令展开`handleHi`的代码：

```shell
(pprof) list handleHi
Total: 2.05GB
ROUTINE ======================== chapter8/sources/go-pprof-optimization-demo/step2.handleHi in chapter8/sources/go-pprof-optimization-demo/step2/demo.go
  840.55MB     2.05GB (flat, cum)   100% of Total
         .          .     17:		http.Error(w, "Optional color is invalid", http.StatusBadRequest)
         .          .     18:		return
         .          .     19:	}
         .          .     20:
         .          .     21:	visitNum := atomic.AddInt64(&visitors, 1)
         .       95MB     22:	w.Header().Set("Content-Type", "text/html; charset=utf-8")
  365.52MB     1.48GB     23:	w.Write([]byte("<h1 style='color: " + r.FormValue("color") +
  475.02MB   486.53MB     24:		"'>Welcome!</h1>You are visitor number " + fmt.Sprint(visitNum) + "!"))
         .          .     25:}
         .          .     26:
         .          .     27:func main() {
         .          .     28:	log.Printf("Starting on port 8080")
         .          .     29:	http.HandleFunc("/hi", handleHi)
(pprof) 
```

通过`list`的输出结果我们可以看到`handleHi`函数的第 23~25 行分配了较多内存(见第一列)。

### 5) 第二次优化(step3)

这里我们进行内存分配的优化方法如下：

- 删除`w.Header().Set`这行调用；
- 使用`fmt.Fprintf`替代`w.Write`。

优化后的`handleHi`代码如下：

```go
// go-pprof-optimization-demo/step3/demo.go
... ...
func handleHi(w http.ResponseWriter, r *http.Request) {
        if !rxOptionalID.MatchString(r.FormValue("color")) {
                http.Error(w, "Optional color is invalid", http.StatusBadRequest)
                return
        }

        visitNum := atomic.AddInt64(&visitors, 1)
        fmt.Fprintf(w, "<html><h1 stype='color: %s'>Welcome!</h1>You are visitor number %d!", r.FormValue("color"), visitNum)
}
... ... 
```

再重新执行性能基准测试收集内存采样数据:

```shell
$go test -v -run=^$ -bench=^BenchmarkHi$ -benchtime=2s -memprofile=mem.prof
goos: darwin
goarch: amd64
pkg: github.com/bigwhite/books/effective-go/chapters/chapter8/sources/go-pprof-optimization-demo/step3
BenchmarkHi
BenchmarkHi-8   	 7090537	       346 ns/op	     173 B/op	       1 allocs/op
PASS
ok  	chapter8/sources/go-pprof-optimization-demo/step3	2.925s 
```

和优化前的数据对比，内存分配次数由`5 allocs/op`降为`1 allocs/op`，每 op 分配的字节数也由`364B`降为`173B`了。

再次通过 pprof 对上面内存采样数据进行分析，查看`BenchmarkHi`中的内存分配情况：

```shell
$go tool pprof step3.test mem.prof
File: step3.test
Type: alloc_space
Entering interactive mode (type "help" for commands, "o" for options)
(pprof) list handleHi
Total: 1.27GB
ROUTINE ======================== chapter8/sources/go-pprof-optimization-demo/step3.handleHi in chapter8/sources/go-pprof-optimization-demo/step3/demo.go
   51.50MB     1.27GB (flat, cum)   100% of Total
         .          .     17:		http.Error(w, "Optional color is invalid", http.StatusBadRequest)
         .          .     18:		return
         .          .     19:	}
         .          .     20:
         .          .     21:	visitNum := atomic.AddInt64(&visitors, 1)
   51.50MB     1.27GB     22:	fmt.Fprintf(w, "<html><h1 stype='color: %s'>Welcome!</h1>You are visitor number %d!", r.FormValue("color"), visitNum)
         .          .     23:}
         .          .     24:
         .          .     25:func main() {
         .          .     26:	log.Printf("Starting on port 8080")
         .          .     27:	http.HandleFunc("/hi", handleHi)
(pprof) 
```

我们看到：照比优化前`handleHi`的内存分配的确有大幅减少(第一列：365MB+475MB -> 51.5MB)。

### 6) 零内存分配(step4)

我们看到`handleHi`的内存分配集中到下面这行代码：

```shell
fmt.Fprintf(w, "<html><h1 stype='color: %s'>Welcome!</h1>You are visitor number %d!", r.FormValue("color"), visitNum) 
```

`fmt.Fprintf`的原型如下：

```shell
$ go doc fmt.Fprintf
func Fprintf(w io.Writer, format string, a ...interface{}) (n int, err error) 
```

我们看到`Fprintf`参数列表中的变长参数都是`interface{}`类型。在前面有关接口类型的章节中我们曾提到过一个接口类型占据两个字(word)，在 64 位架构下，这两个字就是 16 个字节。这意味着我们每次调用 fmt.Fprintf，程序就要为每个变参分配一个占用 16 字节的接口类型变量，然后用传入的类型初始化该接口类型变量，这就是这行代码分配内存较多的原因。

如果我们要实现零内存分配，那么我们可以像下面这样优化代码：

```go
// go-pprof-optimization-demo/step4/demo.go
... ...
var visitors int64 // must be accessed atomically

var rxOptionalID = regexp.MustCompile(`^\d*$`)

var bufPool = sync.Pool{
	New: func() interface{} {
		return bytes.NewBuffer(make([]byte, 128))
	},
}

func handleHi(w http.ResponseWriter, r *http.Request) {
	if !rxOptionalID.MatchString(r.FormValue("color")) {
		http.Error(w, "Optional color is invalid", http.StatusBadRequest)
		return
	}

	visitNum := atomic.AddInt64(&visitors, 1)
	buf := bufPool.Get().(*bytes.Buffer)
	defer bufPool.Put(buf)
	buf.Reset()
	buf.WriteString("<h1 style='color: ")
	buf.WriteString(r.FormValue("color"))
	buf.WriteString("'>Welcome!</h1>You are visitor number ")
	b := strconv.AppendInt(buf.Bytes(), visitNum, 10)
	b = append(b, '!')
	w.Write(b)
} 
```

我们看到这里有几点主要优化：

- 使用`sync.Pool`减少重新分配`bytes.Buffer`的次数；
- 采用预分配底层存储的`bytes.Buffer`拼接输出；
- 使用`strconv.AppendInt`将整型数拼接到`bytes.Buffer`中，`strconv.AppendInt`的实现如下：

```go
// $GOROOT/src/strconv/itoa.go
func AppendInt(dst []byte, i int64, base int) []byte {
        if fastSmalls && 0 <= i && i < nSmalls && base == 10 {
                return append(dst, small(int(i))...)
        }
        dst, _ = formatBits(dst, uint64(i), base, i < 0, true)
        return dst
} 
```

我们看到`AppendInt`内置对10进制数的优化，对于我们的代码而言，这个优化的结果就是没有新分配内存，而是利旧了传入的`bytes.Buffer`的实例，这样代码中`strconv.AppendInt`的返回值变量b即是`bytes.Buffer`实例的底层存储切片。

我们来运行一下最新优化后代码的性能基准测试并采样内存分配性能数据：

```shell
$go test -v -run=^$ -bench=^BenchmarkHi$ -benchtime=2s -memprofile=mem.prof
goos: darwin
goarch: amd64
pkg: chapter8/sources/go-pprof-optimization-demo/step4
BenchmarkHi
BenchmarkHi-8   	10765006	       234 ns/op	     199 B/op	       0 allocs/op
PASS
ok  	chapter8/sources/go-pprof-optimization-demo/step4	2.884s 
```

我们看到：上述性能基准测试的输出结果中每 op 的内存分配次数为 0，同时程序性能也有了提升(346 ns/op -> 234 ns/op)。我们剖析一下输出的内存采样数据：

```shell
$go tool pprof step4.test mem.prof
File: step4.test
Type: alloc_space
Entering interactive mode (type "help" for commands, "o" for options)
(pprof) top
Showing nodes accounting for 2.12GB, 100% of 2.12GB total
      flat  flat%   sum%        cum   cum%
    2.12GB   100%   100%     2.12GB   100%  bytes.makeSlice
         0     0%   100%     2.12GB   100%  bytes.(*Buffer).Write
         0     0%   100%     2.12GB   100%  bytes.(*Buffer).grow
         0     0%   100%     2.12GB   100%  chapter8/sources/go-pprof-optimization-demo/step4.BenchmarkHi
         0     0%   100%     2.12GB   100%  chapter8/sources/go-pprof-optimization-demo/step4.handleHi
         0     0%   100%     2.12GB   100%  net/http/httptest.(*ResponseRecorder).Write
         0     0%   100%     2.12GB   100%  testing.(*B).launch
         0     0%   100%     2.12GB   100%  testing.(*B).runN
(pprof) list handleHi
Total: 2.12GB
ROUTINE ======================== chapter8/sources/go-pprof-optimization-demo/step4.handleHi in chapter8/sources/go-pprof-optimization-demo/step4/demo.go
         0     2.12GB (flat, cum)   100% of Total
         .          .     33:	buf.WriteString("<h1 style='color: ")
         .          .     34:	buf.WriteString(r.FormValue("color"))
         .          .     35:	buf.WriteString("'>Welcome!</h1>You are visitor number ")
         .          .     36:	b := strconv.AppendInt(buf.Bytes(), visitNum, 10)
         .          .     37:	b = append(b, '!')
         .     2.12GB     38:	w.Write(b)
         .          .     39:}
         .          .     40:
         .          .     41:func main() {
         .          .     42:	log.Printf("Starting on port 8080")
         .          .     43:	http.HandleFunc("/hi", handleHi)
(pprof) 
```

我们看到`top`命令排行中`handleHi`排名已经下降了多个位次，并且从`handleHi`代码展开的结果也已经看不到内存分配的数据了(第一列)。

### 7) 查看并发情况下的竞争情况(step5)

前面进行的性能基准测试都是顺序执行的，无法反映出`handleHi`在并发情况下多个goroutine的一些竞争情况，比如在某个处理环节等待时间过长等。为了了解并发情况下`handleHi`的表现，我们为它编写一个并发性能基准测试：

```go
// go-pprof-optimization-demo/step5/demo_test.go
... ...
func BenchmarkHiParallel(b *testing.B) {
        r, err := http.ReadRequest(bufio.NewReader(strings.NewReader("GET /hi HTTP/1.0\r\n\r\n")))
        if err != nil {
                b.Fatal(err)
        }
        b.ResetTimer()

        b.RunParallel(func(pb *testing.PB) {
                rw := httptest.NewRecorder()
                for pb.Next() {
                        handleHi(rw, r)
                }
        })
}
... ... 
```

执行该基准测试，并对阻塞时间类型数据(`block.prof`)进行采样并剖析:

```shell
$go test -bench=Parallel -blockprofile=block.prof      
goos: darwin
goarch: amd64
pkg: chapter8/sources/go-pprof-optimization-demo/step5
BenchmarkHiParallel-8   	15029988	       118 ns/op
PASS
ok  	github.com/bigwhite/books/effective-go/chapters/chapter8/sources/go-pprof-optimization-demo/step5	2.092s

$go tool pprof step5.test block.prof 
File: step5.test
Type: delay
Entering interactive mode (type "help" for commands, "o" for options)
(pprof) top
Showing nodes accounting for 3.70s, 100% of 3.70s total
Dropped 18 nodes (cum <= 0.02s)
Showing top 10 nodes out of 15
      flat  flat%   sum%        cum   cum%
     1.85s 50.02% 50.02%      1.85s 50.02%  runtime.chanrecv1
     1.85s 49.98%   100%      1.85s 49.98%  sync.(*WaitGroup).Wait
         0     0%   100%      1.85s 49.98%  chapter8/sources/go-pprof-optimization-demo/step5.BenchmarkHiParallel
         0     0%   100%      1.85s 50.02%  main.main
         0     0%   100%      1.85s 50.02%  runtime.main
         0     0%   100%      1.85s 50.02%  testing.(*B).Run
         0     0%   100%      1.85s 49.98%  testing.(*B).RunParallel
         0     0%   100%      1.85s 50.01%  testing.(*B).doBench
         0     0%   100%      1.85s 49.98%  testing.(*B).launch
         0     0%   100%      1.85s 50.01%  testing.(*B).run
(pprof) list handleHi
Total: 3.70s
ROUTINE ======================== chapter8/sources/go-pprof-optimization-demo/step5.handleHi in chapter8/sources/go-pprof-optimization-demo/step5/demo.go
         0    18.78us (flat, cum) 0.00051% of Total
         .          .     19:		return bytes.NewBuffer(make([]byte, 128))
         .          .     20:	},
         .          .     21:}
         .          .     22:
         .          .     23:func handleHi(w http.ResponseWriter, r *http.Request) {
         .    18.78us     24:	if !rxOptionalID.MatchString(r.FormValue("color")) {
         .          .     25:		http.Error(w, "Optional color is invalid", http.StatusBadRequest)
         .          .     26:		return
         .          .     27:	}
         .          .     28:
         .          .     29:	visitNum := atomic.AddInt64(&visitors, 1)
(pprof) 
```

我们看到`handleHi`并未出现在top10排名中，进一步展开`handleHi`代码后，我们发现整个函数并没有阻塞 goroutine 过长时间的环节，因此无需对`handleHi`做任何这方面的优化了。当然这也源于 Go 标准库对`regexp`包的`Regexp.MatchString`方法做过的针对并发的优化（也是采用`sync.Pool`），具体优化方法这里就不赘述了。

## 3. 小结

本节要点：

- 通过性能基准测试判定程序是否存在性能瓶颈，如存在瓶颈，可通过 Go 工具链中的`pprof`对程序性能进行剖析；
- 性能剖析分为两个阶段：数据采集和数据剖析；
- `go tool pprof`工具支持多种数据采集方式：通过性能基准测试输出采样结果和独立程序的性能数据采集；
- `go tool pprof`工具支持多种性能数据采样类型：CPU类型(`-cpuprofile`)、堆内存分配类型(`-memprofile`)、锁竞争类型(`-mutexprofile`)、阻塞时间数据类型(`-blockprofile`)等；
- `go tool pprof`支持两种主要的性能数据剖析方式：命令行交互式和基于 Web 的图形化方式；
- 在不明确瓶颈原因情况下，优先对 CPU 类型和堆内存分配类型性能采样数据进行剖析。