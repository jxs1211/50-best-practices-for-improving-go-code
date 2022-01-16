24 sync 包的正确使用姿势

## sync 包的正确使用姿势

Go 语言在提供 **“Communicating Sequential Processes(CSP)”** 并发模型原语的同时，还通过标准库的** sync 包提供了针对传统基于共享内存并发模型**的基本同步原语，包括：互斥锁（sync.Mutex）、读写锁（sync.RWMutex）、条件变量（sync.Cond）等。

## 1. sync 包还是 channel

Go 语言提倡 **“不要通过共享内存来通信，而应该通过通信来共享内存”**。正如在之前的章节中所学的那样，我们建议大家优先使用 CSP 并发模型进行并发程序设计。但是在下面一些场景下，我们依然需要 sync 包提供的低级同步原语。

- 需要高性能的临界区（critical section）同步机制场景

在 Go 中，channel 属于高级同步原语，其自身的实现也是建构在低级同步原语之上的。因此，channel 自身的性能与低级同步原语相比要略微逊色。因此，在需要高性能的临界区（critical section）同步机制的情况下，sync 包提供的低级同步原语更为适合。下面是 sync.Mutex 和 channel 各自实现的临界区同步机制的一个简单性能对比：

```go
// go-sync-package-1_test.go 
package main

import (
	"sync"
	"testing"
)

var cs = 0 // 模拟临界区要保护的数据
var mu sync.Mutex
var c = make(chan struct{}, 1)

func criticalSectionSyncByMutex() {
	mu.Lock()
	cs++
	mu.Unlock()
}

func criticalSectionSyncByChan() {
	c <- struct{}{}
    defer func(){
    	<-c
    }()
	cs++
}

func BenchmarkCriticalSectionSyncByMutex(b *testing.B) {
	for n := 0; n < b.N; n++ {
		criticalSectionSyncByMutex()
	}
}

func BenchmarkCriticalSectionSyncByChan(b *testing.B) {
	for n := 0; n < b.N; n++ {
		criticalSectionSyncByChan()
	}
} 
```

运行这个对比测试(Go 1.13.6)：

```shell
$go test -bench . go-sync-package-1_test.go 
goos: darwin
goarch: amd64
BenchmarkCriticalSectionSyncByMutex-8   	84364287	        13.3 ns/op
BenchmarkCriticalSectionSyncByChan-8    	26449521	        44.4 ns/op
PASS
ok  	command-line-arguments	2.362s 
```

我们看到在这个对比实验中，`sync.Mutex`实现的同步机制的性能要比 channel 实现的高出三倍还多。

- 不想转移结构体对象所有权，但又要保证结构体内部状态数据的同步访问的场景

基于 channel 的并发设计的一个特点就是：在 goroutine 间通过 channel 转移数据对象的所有权。只有拥有数据对象所有权（从 channel 接收到该数据）的 goroutine 才可以对该数据对象进行状态变更。如果你的设计中没有转移结构体对象所有权，但又要保证结构体内部状态数据在多个 goroutine 之间同步访问，那么你可以使用 sync 包提供的低级同步原语来实现，比如最常用的`sync.Mutex`。

### 2. sync 包使用的注意事项

在`$GOROOT/src/sync/mutex.go`文件中，我们看到这样一行关于 sync 包使用的注意事项：

```shell
// Values containing the types defined in this package should not be copied.
// 不应复制那些包含了此包中类型的值 
```

在 sync 包的其他源文件中，我们还会看到类似如下的一些注释：

```shell
 // $GOROOT/src/sync/mutex.go
// A Mutex must not be copied after first use. （禁止复制首次使用后的Mutex）

// $GOROOT/src/sync/rwmutex.go
// A RWMutex must not be copied after first use.（禁止复制首次使用后的RWMutex）

// $GOROOT/src/sync/cond.go
// A Cond must not be copied after first use.（禁止复制首次使用后的Cond）
... ... 
```

为什么不应对 Mutex 等 sync 包中定义的结构类型在首次使用后进行复制操作呢？我们来看一个例子：

```go
// go-sync-package-2.go
package main

import (
	"log"
	"sync"
	"time"
)

type foo struct {
	n int
	sync.Mutex
}

func main() {
	f := foo{n: 17}

	go func(f foo) {
		for {
			log.Println("g2: try to lock foo...")
			f.Lock()
			log.Println("g2: lock foo ok")
			time.Sleep(3 * time.Second)
			f.Unlock()
			log.Println("g2: unlock foo ok")
		}
	}(f)

	f.Lock()
	log.Println("g1: lock foo ok")

	// 在mutex首次使用后复制其值
	go func(f foo) {
		for {
			log.Println("g3: try to lock foo...")
			f.Lock()
			log.Println("g3: lock foo ok")
			time.Sleep(5 * time.Second)
			f.Unlock()
			log.Println("g3: unlock foo ok")
		}
	}(f)

	time.Sleep(1000 * time.Second)
	f.Unlock()
	log.Println("g1: unlock foo ok")
} 
```

运行该示例：

```shell
$go run go-sync-package-2.go 
2020/02/08 21:16:46 g1: lock foo ok
2020/02/08 21:16:46 g2: try to lock foo...
2020/02/08 21:16:46 g2: lock foo ok
2020/02/08 21:16:46 g3: try to lock foo...
2020/02/08 21:16:49 g2: unlock foo ok
2020/02/08 21:16:49 g2: try to lock foo...
2020/02/08 21:16:49 g2: lock foo ok
2020/02/08 21:16:52 g2: unlock foo ok
2020/02/08 21:16:52 g2: try to lock foo...
2020/02/08 21:16:52 g2: lock foo ok
... ... 
```

我们在示例中创建了两个 goroutine：g2 和 g3。示例运行的结果显示：g3 阻塞在加锁操作上了，而 g2 则如预期正常运行。g2 和 g3 的差别就在于 g2 是在互斥锁首次使用之前创建的，而 g3 则是在互斥锁执行完加锁操作并处于锁定状态之后创建的，并且程序在创建 g3 的时候复制了 foo 的实例（包含了 sync.Mutex 的实例）并在之后使用了这个副本。

Go 标准库中 sync.Mutex 的定义如下：

```go
// $GOROOT/src/sync/mutex.go
type Mutex struct {
        state int32
        sema  uint32
} 
```

我们看到 Mutex 的定义非常简单，它由两个字段 state 和 sema 组成：

- state：表示当前互斥锁的状态。
- sema：用于控制锁状态的信号量。

对 Mutex 实例的复制即是两个整型字段的复制。初始情况下，Mutex 的实例处于**Unlocked**状态(state 和 sema 均为 0)。g2 复制了处于初始状态的 Mutex 实例，副本的 state 和 sema 也均为 0，这与 g2 自己定义一个新的 Mutex 实例无异，这也决定了 g2 后续可以按预期正常运行。

后续主程序调用了 Lock 方法，Mutex 的实例变为**Locked**状态（state 字段值为赋值为`sync.mutexLocked`），而此后 g3 创建时恰恰复制了处于**Locked**状态的 Mutex 实例（副本的 state 字段值亦为`sync.mutexLocked`），因此 g3 再对其实例副本调用 Lock 方法将会导致其进入阻塞状态（也是死锁状态，因为没有任何其他机会调用该副本的 Unlock 方法了，并且 Go 不支持递归锁）。

通过上述示例我们直观地看到：那些 sync 包中的类型的实例在首次使用后被复制得到的副本一旦再被使用将导致不可预期的结果，为此在使用 sync 包中的类型的时候，我们推荐通过**闭包**方式或**传递类型实例（或包裹该类型的类型实例）的地址（指针）** 的方式进行，这是使用 sync 包最值得注意的事项。

## 3. 互斥锁（Mutex）还是读写锁（RWMutex）

sync 包提供了两种用于临界区同步的原语：互斥锁（Mutex）和读写锁（RWMutex）。互斥锁（Mutex）是临时区同步原语的**首选**，它常被用来对结构体对象的内部状态、缓存等进行保护，是使用最为广泛的临界区同步原语。相比之下，读写锁颇受“冷落”，但它依然有其存在的道理和适用的场景。

那读写锁（RWMutex）究竟适合在哪种场景下应用呢？我们先通过下面示例来对比一下互斥锁和读写锁在不同并发量下的性能数据：

```go
// go-sync-package-3_test.go 
package main

import (
	"sync"
	"testing"
)

var cs1 = 0 // 模拟临界区要保护的数据
var mu1 sync.Mutex
var cs2 = 0 // 模拟临界区要保护的数据
var mu2 sync.RWMutex

func BenchmarkReadSyncByMutex(b *testing.B) {
	b.RunParallel(func(pb *testing.PB) {
		for pb.Next() {
			mu1.Lock()
			_ = cs1
			mu1.Unlock()
		}
	})
}

func BenchmarkReadSyncByRWMutex(b *testing.B) {
	b.RunParallel(func(pb *testing.PB) {
		for pb.Next() {
			mu2.RLock()
			_ = cs2
			mu2.RUnlock()
		}
	})
}

func BenchmarkWriteSyncByRWMutex(b *testing.B) {
	b.RunParallel(func(pb *testing.PB) {
		for pb.Next() {
			mu2.Lock()
			cs2++
			mu2.Unlock()
		}
	})
} 
```

我们分别在 cpu=2, 8，16，32，64, 128 的情况下运行上述并发性能测试，测试结果如下：

```shell
$go test -bench . go-sync-package-3_test.go -cpu 2  
goos: darwin
goarch: amd64
BenchmarkReadSyncByMutex-2      	72718717	        16.4 ns/op
BenchmarkReadSyncByRWMutex-2    	29053934	        41.2 ns/op
BenchmarkWriteSyncByRWMutex-2   	38043865	        28.7 ns/op
PASS
ok  	command-line-arguments	3.576s

$go test -bench . go-sync-package-3_test.go -cpu 8
goos: darwin
goarch: amd64
BenchmarkReadSyncByMutex-8      	23004751	        52.8 ns/op
BenchmarkReadSyncByRWMutex-8    	29302923	        40.8 ns/op
BenchmarkWriteSyncByRWMutex-8   	19118193	        61.7 ns/op
PASS
ok  	command-line-arguments	3.757s

$go test -bench . go-sync-package-3_test.go -cpu 16 
goos: darwin
goarch: amd64
BenchmarkReadSyncByMutex-16       	20492412	        58.8 ns/op
BenchmarkReadSyncByRWMutex-16     	29786635	        40.9 ns/op
BenchmarkWriteSyncByRWMutex-16    	17095704	        68.1 ns/op
PASS
ok  	command-line-arguments	3.768s

$go test -bench . go-sync-package-3_test.go -cpu 32
goos: darwin
goarch: amd64
BenchmarkReadSyncByMutex-32       	20217310	        63.4 ns/op
BenchmarkReadSyncByRWMutex-32     	29373686	        40.7 ns/op
BenchmarkWriteSyncByRWMutex-32    	14463114	        81.6 ns/op
PASS
ok  	command-line-arguments	3.853s

$go test -bench . go-sync-package-3_test.go -cpu 64
goos: darwin
goarch: amd64
BenchmarkReadSyncByMutex-64       	20733363	        66.1 ns/op
BenchmarkReadSyncByRWMutex-64     	34930328	        34.4 ns/op
BenchmarkWriteSyncByRWMutex-64    	15703741	        82.8 ns/op
PASS
ok  	command-line-arguments	4.057s

$go test -bench . go-sync-package-3_test.go -cpu 128
goos: darwin
goarch: amd64
BenchmarkReadSyncByMutex-128        	19807524	        68.2 ns/op
BenchmarkReadSyncByRWMutex-128      	29254756	        40.8 ns/op
BenchmarkWriteSyncByRWMutex-128     	14505304	        81.8 ns/op
PASS
ok  	command-line-arguments	3.936s 
```

通过测试结果对比，我们得到一些结论：

- 并发量较小的情况下，Mutex 性能最好；随着并发量增大，Mutex 的竞争激烈，导致加锁和解锁性能下降；
- RWMutex 的读锁性能并未随并发量的增大而发生较大变化，性能始终恒定在 40ns 左右；
- 在并发量较大的情况下，RWMutex 的写锁性能与 Mutex、RWMutex 读锁相比是最差的，并且随着并发量增大，写锁性能有继续下降趋势。

由此，我们可以看出，读写锁适合应用在**具有一定并发量且读多写少的场合**。在大量并发读的情况下，多个 goroutine 可以同时持有读锁，从而减少在锁竞争中等待的时间；而互斥锁即便是读请求，同一时刻也只能有一个 goroutine 持有锁，其他 goroutine 只能阻塞在加锁操作上等待被调度。

## 4. 条件变量

`sync.Cond`是传统的条件变量原语概念在 Go 语言中的实现。一个条件变量可以理解为是一个容器，这个容器中存放着一个或一组等待着某个条件成立的 goroutine。当条件成立后，这些处于等待状态的 goroutine 将得到通知并被唤醒继续后续的工作。这与百米飞人大战赛场上各位运动员等待裁判员的发令枪声十分类似。

条件变量是同步原语的一种，如果没有条件变量，开发人员可能需要在 goroutine 中通过连续轮询的方式检查是否满足条件，这种连续轮询非常消耗资源，因为 goroutine 在这个过程中处于活动状态但其工作并无进展。下面就是一个用`sync.Mutex`实现对条件轮询等待的例子：

```go
// go-sync-package-4.go
package main

import (
	"fmt"
	"sync"
	"time"
)

type signal struct{}

var ready bool

func worker(i int) {
	fmt.Printf("worker %d: is working...\n", i)
	time.Sleep(1 * time.Second)
	fmt.Printf("worker %d: works done\n", i)
}

func spawnGroup(f func(i int), num int, mu *sync.Mutex) <-chan signal {
	c := make(chan signal)
	var wg sync.WaitGroup

	for i := 0; i < num; i++ {
		wg.Add(1)
		go func(i int) {
			defer wg.Done()
			for {	
				mu.Lock()
				if !ready {
					mu.Unlock()
					time.Sleep(100 * time.Millisecond)
					continue
				}
				mu.Unlock()
				fmt.Printf("worker %d: start to work...\n", i)
				f(i)
				return
			}
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
	mu := &sync.Mutex{}
	c := spawnGroup(worker, 5, mu)

	time.Sleep(3 * time.Second) // 模拟ready前的准备工作
	fmt.Println("the group of workers start to work...")

	mu.Lock()
	ready = true
	mu.Unlock()

	<-c
	fmt.Println("the group of workers work done!")
} 
```

`sync.Cond`为 goroutine 在上述场景下提供了另一种可选的、资源消耗更小、使用体验更佳的同步方式。使用条件变量原语，我们可以在实现相同目标的同时避免对条件的轮询。

我们用`sync.Cond`对上面的例子进行改造，改造后的代码如下：

```go
// go-sync-package-5.go 
package main

import (
	"fmt"
	"sync"
	"time"
)

type signal struct{}

var ready bool

func worker(i int) {
	fmt.Printf("worker %d: is working...\n", i)
	time.Sleep(1 * time.Second)
	fmt.Printf("worker %d: works done\n", i)
}

func spawnGroup(f func(i int), num int, groupSignal *sync.Cond) <-chan signal {
	c := make(chan signal)
	var wg sync.WaitGroup

	for i := 0; i < num; i++ {
		wg.Add(1)
		go func(i int) {
			groupSignal.L.Lock()
			for !ready {
				groupSignal.Wait()
			}
			groupSignal.L.Unlock()
			fmt.Printf("worker %d: start to work...\n", i)
			f(i)
			wg.Done()
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
	groupSignal := sync.NewCond(&sync.Mutex{})
	c := spawnGroup(worker, 5, groupSignal)

	time.Sleep(5 * time.Second) // 模拟ready前的准备工作
	fmt.Println("the group of workers start to work...")

	groupSignal.L.Lock()
	ready = true
	groupSignal.Broadcast()
	groupSignal.L.Unlock()

	<-c
	fmt.Println("the group of workers work done!")
} 
```

运行该实例：

```shell
$go run go-sync-package-5.go
start a group of workers...
the group of workers start to work...
worker 4: start to work...
worker 4: is working...
worker 1: start to work...
worker 1: is working...
worker 3: start to work...
worker 3: is working...
worker 5: start to work...
worker 5: is working...
worker 2: start to work...
worker 2: is working...
worker 1: works done
worker 3: works done
worker 4: works done
worker 2: works done
worker 5: works done
the group of workers work done! 
```

我们看到`sync.Cond`实例的初始化需要一个满足实现了`sync.Locker`接口的类型实例，通常我们使用`sync.Mutex`。条件变量需要这个互斥锁来同步临界区，保护用作条件的数据。各个等待条件成立的 goroutine 在加锁后判断条件是否成立， 如果不成立，则调用`sync.Cond`的 Wait 方法进入等待状态。Wait 方法在 goroutine 挂起前会进行 Unlock 操作。

当 main goroutine 将`ready`置为 true 并调用`sync.Cond`的 Broadcast 方法后，各个阻塞的 goroutine 将被唤醒并从 Wait 方法中返回。Wait 方法返回前，Wait 方法会再次加锁让 goroutine 进入临界区。接下来 goroutine 会再次对条件数据进行判定，如果条件成立，则解锁并进入下一个工作阶段；如果条件依旧不成立，那么再次调用 Wait 方法挂起等待。

和 sync.Mutex 相比，sync.Cond 应用的场景有限。并且我们在上一节讲解 channel 使用模式时曾使用 channel 承载“信号”实现过一个与`sync.Cond`功能类似的例子（只是少了一个条件判断的步骤），那个例子在代码上还要比使用`sync.Cond`更为清晰简单一些。

## 5. 使用 sync.Once 实现单例（singleton）模式

到目前为止，我们知道的在程序运行期间只被执行一次且 goroutine 安全的函数只有每个包的 init 函数。sync 包提供了另外一种更为灵活的机制，可以保证**任意某个函数**在程序运行期间只被执行一次，这就是**sync.Once**。

在 Go 标准库中，我们看到**sync.Once**的“仅执行一次”语义被一些包用于初始化和资源清理的过程中，以避免重复执行初始化或资源关闭操作。比如：

```go
// $GOROOT/src/mime/type.go
func TypeByExtension(ext string) string {
        once.Do(initMime)
	... ...
}

// $GOROOT/src/io/pipe.go
func (p *pipe) CloseRead(err error) error {
        if err == nil {
                err = ErrClosedPipe
        }
        p.rerr.Store(err)
        p.once.Do(func() { close(p.done) })
        return nil
} 
```

`sync.Once`的语义还是十分适合实现单例模式，并且实现起来十分简单。我们看下面的例子(注意：GetInstance 利用 sync.Once 实现的单例模式本可以十分简单，这里为了后续的说明，我们在例子中的单例函数实现中增加了很多不必要的代码)：

```go
// go-sync-package-6.go
package main

import (
	"log"
	"sync"
	"time"
)

type Foo struct {
}

var once sync.Once
var instance *Foo

func GetInstance(id int) *Foo {
	defer func() {
		if e := recover(); e != nil {
			log.Printf("goroutine-%d: caught a panic: %s", id, e)
		}
	}()
	log.Printf("goroutine-%d: enter GetInstance\n", id)
	once.Do(func() {
		instance = &Foo{}
		time.Sleep(3 * time.Second)
		log.Printf("goroutine-%d: the addr of instance is %p\n", id, instance)
		panic("panic in once.Do function")
	})
	return instance
}

func main() {
	var wg sync.WaitGroup
	for i := 0; i < 5; i++ {
		wg.Add(1)
		go func(i int) {
             defer wg.Done()
			inst := GetInstance(i)
			log.Printf("goroutine-%d: the addr of instance returned is %p\n", i, inst)
		}(i + 1)
	}
	time.Sleep(5 * time.Second)
	inst := GetInstance(0)
	log.Printf("goroutine-0: the addr of instance returned is %p\n", inst)

	wg.Wait()
	log.Printf("all goroutines exit\n")
} 
```

运行该示例：

```shell
$go run go-sync-package-6.go
2020/02/09 18:46:30 goroutine-1: enter GetInstance
2020/02/09 18:46:30 goroutine-4: enter GetInstance
2020/02/09 18:46:30 goroutine-5: enter GetInstance
2020/02/09 18:46:30 goroutine-3: enter GetInstance
2020/02/09 18:46:30 goroutine-2: enter GetInstance
2020/02/09 18:46:33 goroutine-1: the addr of instance is 0x1199b18
2020/02/09 18:46:33 goroutine-1: caught a panic: panic in once.Do function
2020/02/09 18:46:33 goroutine-1: the addr of instance returned is 0x0
2020/02/09 18:46:33 goroutine-4: the addr of instance returned is 0x1199b18
2020/02/09 18:46:33 goroutine-5: the addr of instance returned is 0x1199b18
2020/02/09 18:46:33 goroutine-3: the addr of instance returned is 0x1199b18
2020/02/09 18:46:33 goroutine-2: the addr of instance returned is 0x1199b18
2020/02/09 18:46:35 goroutine-0: enter GetInstance
2020/02/09 18:46:35 goroutine-0: the addr of instance returned is 0x1199b18
2020/02/09 18:46:35 all goroutines exit 
```

通过上述例子，我们观察到：

- [once.Do](http://once.do/) 会等待 f 执行完毕后才返回，这期间其它执行 [once.Do](http://once.do/) 函数的 goroutine(如上面运行结果中的 goroutine 2~5)将会阻塞等待；
- Do 函数返回后，后续的 goroutine 再执行 Do 函数将不再执行 f 并立即返回（如上面运行结果中的 goroutine-0）；
- 即便在函数 f 中出现 panic，sync.Once 原语也会认为 [once.Do](http://once.do/) 执行完毕了，后续对 [once.Do](http://once.do/) 的调用将不再执行 f。

## 6. 使用 sync.Pool 降低 GC 压力

sync 包除了提供像 Mutex 这样的同步原语之外，还针对并发程序的实际需求提供了一些十分实用的工具，比如：`sync.Pool`以及上面讲过的`sync.Once`等。sync.Pool 是一个数据对象缓存池，它具有如下特点：

- 它是 goroutine 并发安全的，可以被多个 goroutine 同时使用；
- 放入该缓存池中的数据对象的生命是暂时的，随时都可能被 GC 回收；
- 缓存池中的数据对象是可以被重复利用的，这样可以一定程度上降低 GC 频繁回收数据对象占用内存的压力。

我们来看一个使用 sync.Pool 分配数据对象与通过 new 等常规方法分配数据对象对比的例子：

```go
// go-sync-package-7_test.go 
package main

import (
	"bytes"
	"sync"
	"testing"
)

var bufPool = sync.Pool{
	New: func() interface{} {
		return new(bytes.Buffer)
	},
}

func writeBufFromPool(data string) {
	b := bufPool.Get().(*bytes.Buffer)
	b.Reset()
	b.WriteString(data)
	bufPool.Put(b)
}
func writeBufFromNew(data string) *bytes.Buffer {
	b := new(bytes.Buffer)
	b.WriteString(data)
	return b
}

func BenchmarkWithoutPool(b *testing.B) {
	b.ReportAllocs()
	for i := 0; i < b.N; i++ {
		writeBufFromNew("hello")
	}
}

func BenchmarkWithPool(b *testing.B) {
	b.ReportAllocs()
	for i := 0; i < b.N; i++ {
		writeBufFromPool("hello")
	}
} 
```

运行这个测试用例：

```shell
$go test -bench . go-sync-package-7_test.go 
goos: darwin
goarch: amd64
BenchmarkWithoutPool-8   	33605625	        32.8 ns/op	      64 B/op	       1 allocs/op
BenchmarkWithPool-8      	53222953	        22.8 ns/op	       0 B/op	       0 allocs/op
PASS
ok  	command-line-arguments	2.385s 
```

我们看到通过`sync.Pool`来复用数据对象的方式可以有效降低内存分配频率，降低 GC 回收压力，从而提高处理性能。`sync.Pool`的一个典型应用就是建立像`bytes.Buffer`这样类型的临时缓存对象池：

```go
var bufPool = sync.Pool{
	New: func() interface{} {
		return new(bytes.Buffer)
	},
} 
```

但实践告诉我们这么用很可能会产生[一些问题](https://github.com/golang/go/issues/23199)。由于 sync.Pool 的 Get 方法从缓存池中挑选 bytes.Buffer 数据对象时并未考虑该数据对象是否满足调用者的需求，因此一旦返回的 Buffer 对象是刚刚被“大数据”撑大后的，并且即将被长期用于处理一些“小数据”时，这个 Buffer 对象所占用的“大内存”将长时间得不到释放。一旦这类情况集中出现，将会给 Go 应用带来沉重的内存消耗负担。为此，目前的 Go 标准库采用两种方式来缓解这一问题。

- 对要放回到缓存池中的数据对象大小做限制

在 Go 标准库 fmt 包中的代码中，我们看到：

```go
// $GOROOT/src/fmt/print.go
func (p *pp) free() {
        // 正确使用sync.Pool要求每个条目需要具有大致相同的内存成本。
        // 若缓存池中存储的类型具有可变大小的缓冲区时，
        // 我们对放回缓存池的对象增加了一个最大缓冲区的硬限制(不能大于65536字节)
        //
        // See https://golang.org/issue/23199
        if cap(p.buf) > 64<<10 {
                return
        }

        p.buf = p.buf[:0]
        p.arg = nil
        p.value = reflect.Value{}
        p.wrappedErr = nil
        ppFree.Put(p)
} 
```

fmt 包对于要 put 回缓存池的 buffer 对象做了一个限制性校验：如果 buffer 的容量大于`64<<10`，则不让其回到缓存池中，这样可以一定程度上缓解处理小对象时重复利用大 Buffer 导致的内存占用问题。

- 建立多级缓存池

标准库的 http 包在处理 http2 数据时，预先建立了多个不同大小的缓存池：

```go
// $GOROOT/src/net/http/h2_bundle.go
var (   
        http2dataChunkSizeClasses = []int{
                1 << 10,
                2 << 10,
                4 << 10,
                8 << 10,
                16 << 10,
        }
        http2dataChunkPools = [...]sync.Pool{
                {New: func() interface{} { return make([]byte, 1<<10) }},
                {New: func() interface{} { return make([]byte, 2<<10) }},
                {New: func() interface{} { return make([]byte, 4<<10) }},
                {New: func() interface{} { return make([]byte, 8<<10) }},
                {New: func() interface{} { return make([]byte, 16<<10) }},
        }
)

func http2getDataBufferChunk(size int64) []byte {
	i := 0
	for ; i < len(http2dataChunkSizeClasses)-1; i++ {
		if size <= int64(http2dataChunkSizeClasses[i]) {
			break
		}
	}
	return http2dataChunkPools[i].Get().([]byte)
}
  
func http2putDataBufferChunk(p []byte) {
	for i, n := range http2dataChunkSizeClasses {
		if len(p) == n {
			http2dataChunkPools[i].Put(p)
			return
		}
	}
	panic(fmt.Sprintf("unexpected buffer len=%v", len(p)))
} 
```

这样我们就可以根据要处理的数据的大小从最适合的缓存池中获取 Buffer 对象并在完成数据处理后将对象归还到对应的池中，而池中的所有临时 buffer 对象的容量始终是保持一致的，这样可以尽量避免“大材小用”的浪费内存的情况。

## 7. 小结

本小节我们对 Go 语言通过 sync 包提供的针对共享内存并发模型的原语的使用方法尤其是注意事项做了细致的说明。下面是本节要点：

- 明确 sync 包中原语应用的适宜场景；
- sync 包内定义的结构体或包含这些类型的结构体在首次使用后禁止复制；
- 明确 sync.RWMutex 的适用场景；
- 掌握条件变量的应用场景和使用方法；
- 实现单例模式时优先考虑 sync.Once；
- 了解 sync.Pool 的优点、使用中可能遇到的问题及解决方法。