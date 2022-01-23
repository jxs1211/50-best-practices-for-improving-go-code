41 与时俱进！使用module管理依赖包

## 与时俱进！使用module管理依赖包

自 2007 年 “三巨头”([Robert Griesemer](https://github.com/griesemer)、[Rob Pike](https://en.wikipedia.org/wiki/Rob_Pike) 和 [Ken Thompson](https://en.wikipedia.org/wiki/Ken_Thompson)) 提出设计和实现 Go 语言以来，[Go 语言](https://tip.golang.org/)已经发展和演化了[十余年](https://tonybai.com/2017/09/24/go-ten-years-and-climbing/)了。这十余年来，Go 取得了巨大的成就，先后在 2009 年和 2016 年当选 [TIOBE](https://www.tiobe.com/tiobe-index/) 年度最佳编程语言，并在全世界范围内拥有数量庞大的拥趸。不过和其他主流编程语言一样，Go 语言也不是完美的，不能满足所有开发者的 “口味”。比如这些年来 Go 在 “包依赖管理” 和 “缺少泛型” 两个方面饱受诟病，Gopher 们希望 Go 核心团队能在这两个方面进行重点改善。

随着 Russ Cox 在 [Go 1.11](https://tip.golang.org/doc/go1.11) 中加入试验性的 go module 机制，Go 语言终于有了原生的包依赖管理机制。经过 [Go 1.12](https://tip.golang.org/doc/go1.12) 到 [Go 1.15](https://tip.golang.org/doc/go1.15) 版本的持续打磨和优化，Go module 机制被越来越多的 Gopher 以及大多数 Go 项目所接受，越来越多的 Go 项目从原先采用 `GOPATH`、`vendor` 机制或是第三方包依赖管理工具 (比如：[dep](https://github.com/golang/dep)、[glide](https://github.com/Masterminds/glide)、[godep](https://github.com/tools/godep) 等) 迁移到使用 go module 来管理包依赖。使用 go module 管理包依赖已经成为 go 项目 **包依赖管理的唯一标准**，并成为高质量 Go 代码的**必要条件**。

## 1. Go 语言包管理演进回顾

为了更好的理解 go module 包依赖管理机制，我们先来看看 Go 语言包管理的演进历史。

### 1.1 go get

Go 在构建设计方面深受 Google 内部开发实践的影响，比如 go get 的设计就深受 [Google 内部单一代码仓库 (single monorepo) 和基于主干 (trunk/mainline based) 的开发模型](https://cacm.acm.org/magazines/2016/7/204032-why-google-stores-billions-of-lines-of-code-in-a-single-repository/pdf)的影响：**只获取 Trunk/mainline 代码和版本无感知**。

![41 与时俱进！使用module管理依赖包](https://img-hello-world.oss-cn-beijing.aliyuncs.com/4efe8563625c135a2c32d5ff8edd7d92.png)

图 10-1-1：Google 内部基于主干的单一仓库模型

Google 内部的这个基于主干的开发模型要求：

- 所有开发人员基于主干 trunk/mainline 开发：提交到 trunk 或从 trunk 获取最新的代码（同步到本地仓库）；
- 版本发布时，建立 Release branch，release branch 实质上就是某一个时刻主干代码的快照；
- 必须同步到 release branch 上的 bug fix 和增强改进代码也通常是先在主干上提交 (commit)，然后再 cherry-pick 到 release branch 上。

基于这个模型，Google 内部各个 project/repository 的 master 分支上的代码都是被认为稳定的 (stable)，Go 语言初期使用的 go get 的行为模式与该模型非常类似：go get 仅仅支持获取 master branch 上的 latest 代码，没有指定 version、branch 或 revision 的能力。

Go 语言新手在初次接触 Go 语言时会感觉到 Go 语言的包获取很方便：只需一行 `go get github.com/user/repo`，github 等代码托管站点上的大量 go 包就可以随你取用。go get 本质上是 [git](https://git-scm.com/)、[hg](https://www.mercurial-scm.org/) 等这些 [版本管理工具](https://en.wikipedia.org/wiki/Version_control)的高级包装。对于使用 git 的 go 包来说，go get 的实质就是将这些包 clone 到本地的特定目录下 (`$GOPATH/src/github.com/user/repo`)，同时 go get 可以自动解析包的依赖，自动下载相关依赖包并调用本地 go 工具链完成包的本地构建。

这种方式在 Google 内部运作良好并不代表在 Google 以外的世界也会被奉为圭皋。渐渐地 Gopher 们从 go get 的 “便利性” 中清醒过来并列出了这样的机制带来的显而易见的问题，至少包括：

- 依赖包持续演进，导致不同 gopher 在不同时间获取和编译你的包时得到的结果可能是不同的，即不能保证可重复的构建 (reproduceable build)；
- 如果依赖包引入了不兼容代码，你的包 / 程序将无法通过编译；
- 如果依赖包因引入新代码而无法正常通过编译，并且该依赖包的作者又未及时修复该问题，这种错误也会传导到你的包，导致你的包无法通过编译。

Gopher 们希望自己项目所依赖的第三方包能受到自己的控制，而不是随意变化。这样 [godep](https://github.com/tools/godep)、[gb](https://getgb.io/)、[glide](https://github.com/Masterminds/glide) 等一批第三方包管理工具便出现了。

以当时 (Go 1.5 版本之前) 应用最为广泛的 [godep](http://tonybai.com/2014/10/30/a-hole-of-godep/) 为例。为了能让第三方依赖包 “稳定下来”，实现项目的可重复构建，godep 将项目当前依赖包的版本信息记录在 `Godeps/Godeps.json` 中，并将依赖包的相关版本存放在 `Godeps/_workspace` 中。

在编译时 (godep go build)，godep 通过临时修改 `GOPATH` 环境变量的方法让 Go 编译器使用缓存在 `Godeps/_workspace` 下的项目依赖的特定版本的第三方包，这样保证了项目不再受制于依赖的第三方包的主分支 (master branch) 上的最新代码变动了。

不过，godep 的 “版本管理” 本质上是通过缓存第三方库的某个 revision 的快照实现的，这种方式依然让人感觉难于管理。同时，通过对 `GOPATH` 的 “偷梁换柱” 的方式实现使用 `Godeps/_workspace` 中的第三方库的快照进行编译也无法使用 Go 原生编译器，项目构建必须使用 `godep go xxx` 来进行。

为此，Go 进一步引入 [vendor 机制](http://tonybai.com/2015/07/10/some-changes-in-go-1-5/) 来**减少 gopher 在包管理问题上的心智负担**。

### 1.2 vendor 机制

Go 核心团队也一直在关注 Go 的包依赖问题，尤其是在 [Go 1.5](http://tonybai.com/2015/07/10/some-changes-in-go-1-5/) 实现自举的情况下，Go 官方依然在 1.5 版本中推出了 [vendor 机制](http://tonybai.com/2015/07/31/understand-go15-vendor/)。`vendor` 机制是 [Russ Cox](https://swtch.com/~rsc/) 在 [Go 1.5 发布](http://tonybai.com/2015/07/10/some-changes-in-go-1-5/)前期以一个试验特性身份紧急加入到 go 中的。vendor 标准化了项目依赖的第三方库的存放位置（不再像 godep 那样需要 `Godeps/_workspace` 了），同时也无需对 `GOPATH` 环境变量进行 “偷梁换柱”，Go 编译器将原生优先感知和使用 `vendor` 目录下缓存的第三方包版本。

不过即便有了 vendor 的支持，vendor 内第三方依赖包的代码的管理依旧是不规范的，要么是手动的，要么是借助 `godep` 这样的第三方包管理工具。自举后的 Go 语言项目本身也引入了 vendor：

```go
// go 1.14
$GOROOT/src $tree -L 3 -F vendor
vendor
├── golang.org/
│   └── x/
│       ├── crypto/
│       ├── net/
│       ├── sys/
│       └── text/
└── modules.txt 
```

不过 go 项目自身对 vendor 中代码的管理方式也是[手动更新](https://github.com/golang/go/commit/554d49af61574bfacd3b106fd5c20aba4a1f1201)，Go 自身并未使用任何第三方的包管理工具。

从 Go 官方角度出发，go 包依赖解决方案的下一步就应该是解决对 vendor 下的第三方包如何进行管理的问题，包括：依赖包的分析、记录和获取等，进而实现项目的可重复构建。Go 社区发起的 `dep` 项目就是用来做这事儿的。

### 1.3 dep

2016 年 [GopherCon 大会](https://www.gophercon.com/)后，在 Go 官方的组织下，一个旨在改善 Go 包管理的委员会（commitee）成立了，共同应对 Go 在包依赖管理上遇到的各种问题。经过各种脑洞和讨论后，该委员会在若干月后发布了 “[包依赖管理技术提案 (Package Management Proposal)](https://groups.google.com/forum/#!topic/go-package-management/P8TehVoFLjg)”，并启动了最有可能被接纳为官方包管理工具的项目 [dep](https://github.com/golang/dep) 的设计和开发。2017 年年初，dep 项目正式对外开放。在 2017 年 5 月，dep 发布了 [v0.1.0 版本](https://github.com/golang/dep/releases/tag/v0.1.0)，并进入 alpha 测试阶段。

Go 包管理委员会的牵头人物是微服务框架 [go-kit](https://github.com/go-kit/kit) 作者 [Peter Bourgon](https://github.com/peterbourgon)，但主导 dep 开发的是 [Sam Boyer](https://github.com/sdboyer)，他也是 dep 底层包依赖分析引擎 [gps](https://github.com/sdboyer/gps) 的作者。

和其他一些第三方 Go 包管理工具有所不同，dep 在进行大规模积极开发之前是经过委员会深思熟虑的，包括：[工具特性](https://docs.google.com/document/d/1JNP6DgSK-c6KqveIhQk-n_HAw3hsZkL-okoleM43NgA/edit)、[用户故事](https://docs.google.com/document/d/1wT8e8wBHMrSRHY4UF_60GCgyWGqvYye4THvaDARPySs/edit)等都在事前做了初步设计。如果你拜读这些文档，你可能会觉得解决包依赖问题，还是[蛮复杂的](https://research.swtch.com/version-sat)。不过，对于这个工具的使用者来说，我们面对的是一些十分简化的交互接口。

dep 总体上参考了当今主流编程语言解决包依赖问题的主流思路：

- 利用包依赖分析引擎 gps 分析当前项目代码中的包依赖关系；
- 将分析出的项目包的直接依赖和约束写入项目根目录下的 **Gopkg.toml** 文件中；
- 将项目依赖的所有第三方包（包括直接依赖和间接依赖 / 传递依赖）在满足 **Gopkg.toml** 中约束范围内的最新版本信息写入 **Gopkg.lock** 文件中；
- 以 **Gopkg.lock** 为输入，将其中的包 (精确到某次 commit 版本) 下载到项目根目录下的 `vendor` 路径下面。

但就像这种思路的局限一样，dep 也不能很好解决类似下面这种 “钻石依赖” 问题：

![41 与时俱进！使用module管理依赖包](https://img-hello-world.oss-cn-beijing.aliyuncs.com/a2e26130231e6bdf81dbcb6e58ddf227.png)

图 10-1-2：“钻石形” 包依赖关系

从图中我们看到：包 foo 依赖 a 和 b 两个包，而 a、b 两个包分别依赖包 f 的不同版本。在这种情况下，由于 a 依赖的 **v1.1.0** 版本 f 和 b 依赖的 **v2.0.0** 版本 f 两个约束之间没有交集，无法调和，dep 将会因无法解决这个依赖冲突而报错。

这一问题背后还有一层原因，那就是 dep 的设计要求[平坦的 vendor](https://blog.gopheracademy.com/advent-2016/saga-go-dependency-management/)，即使用 dep 的项目只能有一个根目录下的 vendor 目录，所以直接依赖或传递依赖的包中包含 vendor 的，vendor 目录也都会 [被 dep 删除掉](https://blog.gopheracademy.com/advent-2016/saga-go-dependency-management/)。这样一旦依赖包中存在带有冲突的约束，那么 dep 必将报错！

dep 从诞生那天起就被 gopher 社区视为最可能成为 Go 官方包管理工具的候选者。由于 dep 的这一 “特殊身份”，虽然 dep 当时离成熟尚远，但 dep 的开发和演进吸引了诸多 gopher 的目光，很多组织已经开始将自己项目的包管理工具迁移为 dep，并为 dep 进行早期测试。dep 项目本身也挪到了 `github.com/golang` 组织的下面，“转正” 看起来只是时间问题。

### 1.4 vgo (go module 的前身)

2018 年初，正当广大 gopher 们认为 **dep** 将 “顺理成章” 地升级为 go 官方工具链的一部分的时候，Go 核心团队的技术负责人，也是 Go 核心团队早期成员之一的 [Russ Cox](https://research.swtch.com/) 在 [个人博客](https://research.swtch.com/)上连续发表了[七篇文章](https://research.swtch.com/vgo)，系统阐述了 Go 团队解决 “包依赖管理” 的技术方案: **[vgo](https://github.com/golang/vgo)**。

vgo 的主要思路包括：[语义导入版本 (Semantic Import Versioning)](https://research.swtch.com/vgo-import)、 [最小版本选择 (Minimal Version Selection)](https://research.swtch.com/vgo-mvs) 和 [引入 Go module 概念](https://research.swtch.com/vgo-module)等。这七篇文章的发布引发了 Go 社区激烈地争论，尤其是 MVS （最小版本选择）与目前主流的依赖版本选择方法的相悖以及在包导入路径上引入版本号让很多传统 Go 包管理工具的维护者 “不满”，当然也包括 “准官方包管理工具” [dep](https://tonybai.com/tag/dep) 的作者和拥趸们。

2018 年 5 月份，Russ Cox 的技术提案 [“cmd/go: add package version support to Go toolchain”](https://github.com/golang/go/issues/24301) 被接纳 (accept)，后 Russ Cox [将 vgo 的代码合并到 Go 项目主干](https://github.com/golang/go/commit/f7248f05946c1804b5519d0b3eb0db054dc9c5d6)，并将这套机制正式命名为 “ **go module**”。由于 vgo 项目本身就是一个实验原型项目，合并到主干后，**vgo 这个术语以及 vgo 项目的使命也就就此结束了**。后续 Go module 机制将直接在 Go 项目主干上继续演化。Go module 的诞生也意味着 dep 项目的生命周期的结束。

## 2. Go module：Go 包依赖管理的生产标准

### 2.1 go module 定义以及 “包依赖管理” 的工作模式

我们在 `source/go-module` 下建立 hello 目录 (注意：此时 `$GOPATH=~/go`，显然 hello 目录并不在 `GOPATH` 下面)。`hello.go` 的代码如下：

```go
// hello.go
package main

import "bitbucket.org/bigwhite/c"

func main() {
    c.CallC()
} 
```

在 `GO111MODULE="off"` 的前提下，我们构建一下 `hello.go` 这个源码文件：

```go
# go build hello.go
$go run hello.go 
hello.go:3:8: cannot find package "bitbucket.org/bigwhite/c" in any of:
	$GOROOT/src/bitbucket.org/bigwhite/c (from $GOROOT)
	/Users/tonybai/Go/src/bitbucket.org/bigwhite/c (from $GOPATH) 
```

我们看到构建错误！错误原因很明了：在本地的 `GOPATH` 下并没有找到 `bitbucket.org/bigwhite/c` 路径下的包 c。传统 fix 这个问题的方法是手工将包 c 通过 `go get` 下载到本地，并且 `go get` 会自动下载包 c 所依赖的包 d：

```go
$ go get bitbucket.org/bigwhite/c
$ go run hello.go
call C: master branch
   --> call D:
    call D: master branch
   --> call D end 
```



这种传统的，也是我们最熟悉的 Go 编译器从 `$GOPATH` 下 (以及 `vendor` 目录下) 搜索目标程序依赖包的模式称为 “**gopath mode**”。

`GOPATH` 是 Go 早期设计的产物，在 Go 语言快速发展的今天，人们日益发现 `GOPATH` 似乎不那么重要了，尤其是在引入 `vendor` 机制以及诸多包管理工具之后。并且 `GOPATH` 的设置还会让 Go 语言新手感到些许困惑，提高了入门的门槛。

Go 核心团队也一直在寻求 “去 GOPATH” 的方案，当然这一过程是循序渐进的。[Go 1.8](https://tonybai.com/2017/02/03/some-changes-in-go-1-8/) 版本中，如果开发者没有显式设置 `GOPATH`，Go 会赋予 `GOPATH` 一个默认值 (比如：在 linux 上为 `$HOME/go`)。虽说不用再设置 `GOPATH`，但 `GOPATH` 还是事实存在的，它在 go 工具链中依旧发挥着至关重要的作用。

Go module 的引入在 “去 GOPATH” 之路上更进了一步，它引入了一种新的依赖管理工作模式：**“module-aware mode”**。在该模式下，通常一个仓库的顶层目录下会放置一个 `go.mod` 文件，每个 `go.mod` 文件唯一定义了一个 module。

**一个 module 就是由一组相关包组成的一个独立的版本单元**。module 是有版本的，module 下的包也就有了版本属性。而放置 `go.mod` 文件的目录被称为 `module root` 目录。module root 目录以及其子目录下的所有 Go 包均归属于该 module，除了那些自身包含 `go.mod` 文件的子目录。虽然 Go 支持在一个仓库 (repo) 中定义多个 module，但**通常的 Go 惯用法是一个仓库就定义一个 module**。

在一个仓库中定义多个 module 的用法**严重不建议使用**，这不仅会给你自己带来麻烦，也很大可能会让你的 module 的使用者感到困惑。

在 “module-aware 模式” 下，Go 编译器将不会在 `GOPATH` 下面以及 `vendor` 下面搜索目标程序依赖的第三方 Go 包。我们来看一下在 “module-aware 模式” 下 `hello.go` 的构建过程：

我们首先在 `hello` 目录下创建 `go.mod`:

```go
// go.mod
module hello 
```

然后构建 `hello.go`：

```go
$go build hello.go
go: finding bitbucket.org/bigwhite/d v0.0.0-20180714005150-3e3f9af80a02
go: finding bitbucket.org/bigwhite/c v0.0.0-20180714063616-861b08fcd24b
go: downloading bitbucket.org/bigwhite/c v0.0.0-20180714063616-861b08fcd24b
go: downloading bitbucket.org/bigwhite/d v0.0.0-20180714005150-3e3f9af80a02

$./hello
call C: master branch
   --> call D:
    call D: master branch
   --> call D end 
```

我们看到 Go 编译器并没有去使用之前已经下载到 `GOPATH` 下的 `bitbucket.org/bigwhite/c` 包和 `bitbucket.org/bigwhite/d` 包，而是重新下载了这两个包并成功编译。我们看看执行 `go build` 后 `go.mod` 文件的内容：

```go
$cat go.mod
module hello

require (
    bitbucket.org/bigwhite/c v0.0.0-20180714063616-861b08fcd24b
    bitbucket.org/bigwhite/d v0.0.0-20180714005150-3e3f9af80a02 // indirect
) 
```

我们看到 Go 编译器分析出了 `hello` module 的依赖包，将其写入 `go.mod` 的 `require` 区域。由于 c、d 两个包均没有发布版本 (建立其他分支或打标签)，因此 Go 编译器使用了包 c 和 d 的当前最新版，并以伪版本 (Pseudo-versions) 的形式作为这两个包的当前版本号。

并且我们看到：`hello` module 并没有直接依赖包 d，并且 `bitbucket.org/bigwhite/c` 下没有建立 `go.mod` 记录包 c 的依赖，因此在 `d` 包的记录后面通过注释形式标记了 `indirect`，即间接依赖。

在 “module-aware 模式” 下，Go 编译器将下载的依赖包缓存在 `$GOPATH/pkg/mod` 下面：

```go
// $GOPATH/pkg/mod
# tree -L 3
.
├── bitbucket.org
│   └── bigwhite
│       ├── c@v0.0.0-20180714063616-861b08fcd24b
│       └── d@v0.0.0-20180714005150-3e3f9af80a02
├── cache
│   ├── download
│   │   ├── bitbucket.org
│   │   ├── golang.org
│   │   └── rsc.io
│   └── vcs
│       ├── 064503657de46d4574a6ab937a7a3b88fee03aec15729f7493a3dc8e35cc6d80
│       ├── 064503657de46d4574a6ab937a7a3b88fee03aec15729f7493a3dc8e35cc6d80.info
│       ├── 0c8659d2f971b567bc9bd6644073413a1534735b75ea8a6f1d4ee4121f78fa5b
... ... 
```

我们看到 c、d 两个包也是按照 “版本” 进行缓存的，便于后续在 “module-aware mode” 下进行包构建使用。

Go module 机制在 Go 1.11 版本中是试验特性，按照 Go 的惯例，在新的试验特性首次加入时，都会有一个特性开关，go module 也不例外，**GO111MODULE** 这个临时的环境变量就是 go module 特性的试验开关。**GO111MODULE** 有三个值：**auto**、**on** 和 **off**，默认值为 **auto**。**GO111MODULE** 的值会直接影响 Go 编译器的 “包依赖管理” 工作模式的选择：是 `gopath mode` 还是 `module-aware mode`。并且随着试验特性的成熟，新版本 Go 会更新 **GO111MODULE** 在不同值下的行为模式，我们来详细看一下。

在 Go 1.11 版本中，**GO111MODULE** 的值对 “包依赖管理” 工作模式的选择以及行为模式如下：

- 当 **GO111MODULE** 的值为 **off** 时，go module 试验特性关闭，Go 编译器会始终使用 **gopath mode**，即无论要构建的源码目录是否在 `GOPATH` 路径下，Go 编译器都会在传统的 `GOPATH` 和 `vendor` 目录下搜索目标程序依赖的 go 包；
- 当 **GO111MODULE** 的值为 **on** 时 (`export GO111MODULE=on`)，go module 试验特性始终开启，Go 编译器会始终使用 **module-aware mode**，即不管要构建的源码目录是否在 `GOPATH` 路径下，Go 编译器都**不会**在传统的 `GOPATH` 和 `vendor` 目录下搜索目标程序依赖的 go 包，而是在 go module 的缓存目录 (默认 `$GOPATH/pkg/mod`) 下搜索对应版本的依赖包；
- 当 **GO111MODULE** 的值为 **auto** 时 (不显式设置即为 **auto**)，也就是我们在上面的例子中所展现的那样：使用 `gopath mode` 还是 `module-aware mode`，取决于要构建的源码目录所在位置以及是否包含 `go.mod` 文件。如果要构建的源码目录不在以 `GOPATH/src` 为根的目录体系下且包含 `go.mod` 文件 (两个条件缺一不可)，那么 Go 编译器将使用 `module-aware mode`；否则使用传统的 `gopath mode`。

在 Go 1.11 中，为了获取一个 module 下的包，我们需要显式地创建一个 `go.mod` 文件，否则我们就会得到类似这样的错误：

```go
//go 1.11.2

# go get github.com/bigwhite/gocmpp
go: cannot find main module; see 'go help modules'

或

# go get github.com/bigwhite/gocmpp
go: cannot determine module path for source directory /Users/tony/test/go (outside GOPATH, no import comments) 
```

这显然非常不方便。Go 1.12 版本对该问题进行了优化：当 **GO111MODULE=on** 时，获取 go module 无需再显式创建一个 `go.mod` 文件了。

在 Go 1.13 版本中，`module-aware mode` 的优先级得到了提升，虽然 **GO111MODULE** 的默认值依然为 `auto`，但 `auto` 值下 Go 编译器的行为模式发生了变化：无论是在 `GOPATH/src` 下还是 `GOPATH` 之外的仓库中，只要目录下有 `go.mod`，Go 编译器都会使用 `module-aware mode` 来管理包依赖。

Go 1.14 版本中，go module 的运作机制、命令及其参数形式、行为特征已趋稳定，可用于生产环境了。**GO111MODULE** 的值对 “包依赖管理” 工作模式的选择以及行为模式变动如下：

- 在 `module-aware mode` 下，如果 `go.mod` 中 go version 是 go 1.14 及以上，且当前仓库顶层目录下有 `vendor` 目录，那么 go 工具链将默认使用 `vendor`(即 `-mod=vendor`) 中的包，而不是 module cache 中的 (`$GOPATH/pkg/mod` 下)。

  同时在这种模式下，go 工具会校验 `vendor/modules.txt` 与 `go.mod` 文件以确保它们保持同步； 在此模式下，如要非要使用 module cache 中的包进行构建，则需要为 go 工具链显式传入 `-mod=mod` ，比如：`go build -mod=mod ./...`。

- 在 `module-aware mode` 下，如果没有建立 `go.mod` 或 go 工具链无法找到 `go.mod`，那么你必须显式传入要处理的 go 源文件列表，否则 go 工具链将需要你明确建立 `go.mod`。比如：在一个没有 `go.mod` 的目录下，要编译一个 `hello.go`，我们需要使用 `go build hello.go`，即 hello.go 需要显式放在命令后面。如果你执行 `go build .`，就会得到类似下面错误信息：

```go
$go build .
go: cannot find main module, but found .git/config in /Users/tonybai
    to create a module there, run:
    cd .. && go mod init 
```

也就是说在没有 `go.mod` 的情况下，go 工具链的功能是受限的。

### 2.2 go module 的依赖包版本的选择

#### build list 和 main module

`go.mod` 文件一旦创建，它的内容就会被 go 工具链全面掌控。go 工具链会在各类命令执行时维护 `go.mod` 文件，比如：`go get`、`go build`、`go mod` 等。

之前的例子中，`hello` module 依赖的 c 和 d (间接依赖) 两个包均没有显式的版本信息，因此 go mod 使用伪版本 (Pseudo-versions) 机制来生成和记录 c 和 d 包的 “版本”，我们可以通过下面命令查看到这些信息：

```shell
$go list -m -json all
{
	"Path": "hello",
	"Main": true,
	"Dir": "sources/go-module/hello",
	"GoMod": "sources/go-module/hello/go.mod",
	"GoVersion": "1.14"
}
{
	"Path": "bitbucket.org/bigwhite/c",
	"Version": "v0.0.0-20180714063616-861b08fcd24b",
	"Time": "2018-07-14T06:36:16Z",
	"Dir": "/Users/tonybai/Go/pkg/mod/bitbucket.org/bigwhite/c@v0.0.0-20180714063616-861b08fcd24b"
	"GoMod": "/Users/tonybai/Go/pkg/mod/cache/download/bitbucket.org/bigwhite/c/@v/vv0.0.0-20180714063616-861b08fcd24b.mod"
}
{
	"Path": "bitbucket.org/bigwhite/d",
	"Version": "v0.0.0-20180714005150-3e3f9af80a02",
	"Time": "2018-07-14T00:51:50Z",
	"Indirect": true,
	"Dir": "/Users/tonybai/Go/pkg/mod/bitbucket.org/bigwhite/d@v0.0.0-20180714005150-3e3f9af80a02",
	"GoMod": "/Users/tonybai/Go/pkg/mod/cache/download/bitbucket.org/bigwhite/d/@v/v0.0.0-20180714005150-3e3f9af80a02.mod"
} 
```

`go list -m` 输出的信息被称为 **build list**，也就是构建当前 module 所需的所有相关包信息的列表。在输出信息中我们看到 `"Main": true` 这一行信息，标识当前的 module 为 **main module**。所谓 **main module**，即是 go build 命令执行时所在当前目录所归属的那个 module，go 命令会在当前目录、当前目录的父目录、父目录的父目录… 等下面寻找 `go.mod` 文件，所找到的第一个 `go.mod文件`对应的 module 即为 **main module**。如果没有找到 `go.mod`，go 命令会提示下面错误信息：

```go
$go build test/hello/hello.go
go: cannot find main module root; see 'go help modules' 
```

当然我们也可以使用下面命令简略输出 **build list**：

```
$go list -m all
hello
bitbucket.org/bigwhite/c v0.0.0-20180714063616-861b08fcd24b
bitbucket.org/bigwhite/d v0.0.0-20180714005150-3e3f9af80a02 
```

#### go.mod 中的 “require”

现在我们通过打标签的方式赋予 c 和 d 这两个包以版本信息：

```
包c:

v1.0.0
v1.1.0
v1.2.0

包d:

v1.0.0
v1.1.0
v1.2.0
v1.3.0 
```

然后我们清除掉 `$GOPATH/pkg/mod` 目录下的内容 (可用 `go clean -modcache` 命令)，并将 `go.mod` 重新置为初始状态，即只包含 module 字段。接下来，我们再来构建一次 `hello.go`:

```go
// sources/go-module/hello目录下

$go build hello.go
go: finding bitbucket.org/bigwhite/c v1.2.0
go: downloading bitbucket.org/bigwhite/c v1.2.0
go: finding bitbucket.org/bigwhite/d v1.3.0
go: downloading bitbucket.org/bigwhite/d v1.3.0

$./hello
call C: v1.2.0
   --> call D:
    call D: v1.3.0
   --> call D end

$cat go.mod
module hello

require (
    bitbucket.org/bigwhite/c v1.2.0 
    bitbucket.org/bigwhite/d v1.3.0 // indirect
) 
```

我们看到，再一次初始构建 hello module 时，Go 编译器不再用伪版本号 (Pseudo-version) 对应的最新提交版本，而是使用了 c 和 d 两个包的最新发布版本：包 c 的 v1.2.0 版本和包 d 的 v1.3.0 版本。

如果我们对使用的 c 和 d 版本有特殊的约束，比如：我们使用包 c 的 v1.0.0 版本和包 d 的 v1.1.0 版本，我们可以通过 `go mod -require` 来显式更新 `go.mod` 文件中的 **require** 段的信息：

```go
$go mod -require=bitbucket.org/bigwhite/c@v1.0.0
$go mod -require=bitbucket.org/bigwhite/d@v1.1.0

$cat go.mod
module hello

require (
    bitbucket.org/bigwhite/c v1.0.0
    bitbucket.org/bigwhite/d v1.1.0 // indirect
)

$go build hello.go
go: finding bitbucket.org/bigwhite/d v1.1.0
go: finding bitbucket.org/bigwhite/c v1.0.0
go: downloading bitbucket.org/bigwhite/c v1.0.0
go: downloading bitbucket.org/bigwhite/d v1.1.0

$./hello
call C: v1.0.0
   --> call D:
    call D: v1.1.0
   --> call D end 
```

我们看到由于我们显式地修改了对 c 和 d 两个包的版本依赖约束，go build 构建时会去下载包 c 的 v1.0.0 和包 d 的 v1.1.0 版本并完成构建。

除了通过传入 `package@version` 给 `go mod -requirement` 来精确 “指示” module 的依赖约束之外，go mod 还支持 `query表达式`，比如：

```
$go mod -require='bitbucket.org/bigwhite/c@>=v1.1.0' 
```

go mod 命令会对 query 表达式做求值，得出 **build list** 使用的包 c 的版本:

```go
$cat go.mod
module hello

require (
    bitbucket.org/bigwhite/c v1.1.0
    bitbucket.org/bigwhite/d v1.1.0 // indirect
)

$go build hello.go
go: downloading bitbucket.org/bigwhite/c v1.1.0

$./hello
call C: v1.1.0
   --> call D:
    call D: v1.1.0
   --> call D end 
```

go mod 命令对 query 表达式进行求值的算法是 “选择最接近于比较目标的版本 (tagged version)”。以上面例子为例：

```go
query text: >=v1.1.0
比较的目标版本为v1.1.0
比较形式：>= 
```

因此，满足这一 query 表达式的最接近于比较目标的版本 (tagged version) 就是 **v1.1.0**。

如果我们给包 d 增加一个约束：“小于 v1.3.0”，我们再来看看 go mod 的选择：

```go
$go mod -require='bitbucket.org/bigwhite/d@<v1.3.0'
$cat go.mod
module hello

require (
    bitbucket.org/bigwhite/c v1.1.0 
    bitbucket.org/bigwhite/d <v1.3.0
)

$go build hello.go
go: finding bitbucket.org/bigwhite/d v1.2.0
go: downloading bitbucket.org/bigwhite/d v1.2.0

$./hello
call C: v1.1.0
   --> call D:
    call D: v1.2.0
   --> call D end 
```

我们看到 go mod 选择了包 d 的 v1.2.0 版本，根据 query 表达式的求值算法，v1.2.0 恰是最接近于 “小于 v1.3.0” 的标签版本。

用下面这幅示意图来呈现这一算法更为直观一些：

![41 与时俱进！使用module管理依赖包](https://img-hello-world.oss-cn-beijing.aliyuncs.com/3ce8b2aba319d58292c65b02cd66a429.png)

图 10-1-3：query 表达式的求值过程

#### 最小版本选择 (minimal version selection, mvs)

每个[依赖管理解决方案](https://tonybai.com/2019/09/21/brief-history-of-go-package-management/)都必须解决选择依赖项版本的问题。当前其他主流语言以及 go module 之前存在的 go 包依赖管理工具选择的算法都试图识别任何依赖项的 “最新最大 (latest greatest)” 版本。在[语义版本控制 (sematic versioning)](https://semver.org/) 被正确应用并且得到遵守的情况下，这是有道理的。在这样的情况下，依赖项的 “最新最大” 版本应该是最稳定和安全的版本，并且应与较早版本具有向后兼容性。至少在相同的主版本 (major verion) 依赖树中是如此。

Go 则采用了最小版本选择 (Minimal Version Selection, MVS) 算法。从本质上讲，Go 团队相信 MVS 为 Go 程序实现持久的和可重复的构建提供了最佳的方案。到目前为止，我们所举的示例都比较简单，hello module 所依赖的包 c 和包 d 也没有使用 `go.mod` 记录自己的依赖。对于复杂的包依赖场景，Go 核心团队的 Russ Cox 在 [“Minimal Version Selection”](https://research.swtch.com/vgo-mvs) 一文中对 Go 编译器在选择依赖 module 版本时所采用的 **最小版本选择**算法做过形象的解释。

![41 与时俱进！使用module管理依赖包](https://img-hello-world.oss-cn-beijing.aliyuncs.com/08846df3353941083eff1f7477527495.png)

图 10-1-4：复杂包依赖最小版本选择的场景

![41 与时俱进！使用module管理依赖包](https://img-hello-world.oss-cn-beijing.aliyuncs.com/27daa26bd4f987e4692510cf59ee3970.png)

图 10-1-5：最小版本选择的算法解释

- 最小版本选择 (mvs) 以 **build list** 为中心，从一个空的 **build list** 集合开始，先加入 main module (A1)，然后递归计算 main module 的 **build list**；

- main module (A1) 的一个直接依赖是包 B。包 B 的所有版本包括：v1.1 和 v1.2。A1 的 go.mod 明确指明依赖的是包 B 的 v1.2 版本，并且 v1.2 已经是包 B 的最新版本，于是选择包 B v1.2；

- 包 B v1.2 依赖包 D，包 D 的所有版本包括：v1.1、v1.2、v1.3 和 v1.4。包 B v1.2 的 go.mod 明确指明依赖的是包 D 的 v1.3 版本，那么 Go 编译器究竟会选择哪个版本的包 D 呢？确实有两种选择。首选是选择 “最新的” 版本，即 v1.4。

  第二个选择是选择包 B v1.2 所需的版本 v1.3。像 dep 这样的依赖工具将选择 v1.4 版，并在语义版本化和遵守社会契约的前提下可以正常工作。但是采用了 go module 机制的 Go 编译器的 MVS 算法会尊重包 B v1.2 的要求并选择包 D 的 v1.3 版本，即在包 B v1.2 的依赖项 (包 D) 的当前所有版本集合中，Go 会选择满足包 B v1.2 要求的 “最小” 版本。同理，包 D v1.3 依赖包 E，Go 编译器同样选择了满足包 D v1.3 版本要求的包 E 的最小版本：v1.2。这样 main module 在包 B 这个直接依赖项上的 **build list** 就浮现了出来：[B v1.2, D v1.3, E v1.2]；

- main module (A1) 的另一个直接依赖是包 C。按照对包 B 的 **build list** 的分析，我们可以得出 main module 在包 C 这个直接依赖项上的 **build list**：[C v1.2, D v1.4, E v1.2]；

- 接下来，Go 编译器会将包 B 和包 C 的 **build list** 去重并合并，形成 **rough build list**：[A1, B v1.2, C v1.2, D v1.3, D v1.4 和 E v1.2]；

- 在这个过程中，我们看到两个 **build list** 中都有包 D 但版本不同。按照[语义化版本规范](https://semver.org/)，包 D 的 v1.3 和 v1.4 两个版本的主版本号 (major) 相同，因此这两个版本是兼容的。为了同时满足包 B 和包 C 的依赖约束，Go 编译器将选择包 D 的 v1.4 版本，这也是可以同时满足包 B 和包 C 的依赖约束的最小版本 (如果包 D 有 v1.5、v1.6 版本亦是如此)。

我们改造一下我们的例子，让它变得复杂些！首先，我们为包 c 添加 `go.mod` 文件，并为其打一个新版本：**v1.3.0**。在包 c 对应的 `go.mod` 文件中，我们为其添加一个依赖约束: `bitbucket.org/bigwhite/d@v1.2.0`。

```go
//bitbucket.org/bigwhite/c/go.mod
module bitbucket.org/bigwhite/c

require (
        bitbucket.org/bigwhite/d v1.2.0
) 
```

接下来，我们将 hello module 重置为初始状态，并清空 module cache (`$GOPATH/pkg/mod目录`下)。我们修改一下 hello module 的 `hello.go` 如下：

```go
// source/go-module/hello/hello.go
package main

import "bitbucket.org/bigwhite/c"
import "bitbucket.org/bigwhite/d"

func main() {
    c.CallC()
    d.CallD()
} 
```

我们让包 d 成为 hello module 的直接依赖，并在其 `go.mod` 中增加关于包 d 的版本约束：

```go
// source/go-module/hello/go.mod
module hello

require (
    bitbucket.org/bigwhite/d v1.3.0
) 
```

我们再来构建一下 hello module：

```go
$go build hello.go
go: finding bitbucket.org/bigwhite/d v1.3.0
go: downloading bitbucket.org/bigwhite/d v1.3.0
go: finding bitbucket.org/bigwhite/c v1.3.0
go: downloading bitbucket.org/bigwhite/c v1.3.0
go: finding bitbucket.org/bigwhite/d v1.2.0

$cat go.mod
module hello

require (
    bitbucket.org/bigwhite/c v1.3.0 
    bitbucket.org/bigwhite/d v1.3.0
)

$./hello
call C: v1.3.0
   --> call D:
    call D: v1.3.0
   --> call D end
call D: v1.3.0 
```

我们看到 Go 编译器按照 “最小版本选择” 算法最终选择了包 d 的 v1.3.0 版本。这里也模仿 Russ Cox 的图解给出 hello module 的 mvs 解析示意图：

![41 与时俱进！使用module管理依赖包](https://img-hello-world.oss-cn-beijing.aliyuncs.com/29677854d881895bc7fc0a3f146f50e8.png)

图 10-1-6：hello module 的 mvs 分析示意图

#### 依赖一个包的不同版本

按照语义化版本规范，当代码演化出现与之前版本的不兼容性变化时，需要升级版本中的 `major` 版本号。而 go module 允许在包导入路径中带有 major 版本号，比如：`"import github.com/user/repo/v2"`，表示所用的包为 v2 版本下的实现。我们甚至可以在一个项目中同时依赖同一个包的不同版本。我们依旧使用上面的例子来实操一下如何在 hello module 中使用包 d 的两个版本的代码。

我们首先需要为包 d 建立 module 文件：`go.mod`，并标识出当前的 module 为 `bitbucket.org/bigwhite/d/v2`（为了保持与 v0/v1 各自独立演进，可通过建立 branch 的方式来实现，然后基于该版本打 v2.0.0 标签）。

```go
// bitbucket.org/bigwhite/d
$cat go.mod
module bitbucket.org/bigwhite/d/v2 
```

改造一下 hello module，这次我们导入包 d 的 v2 版本：

```go
// sources/go-module/hello/hello.go
package main

import "bitbucket.org/bigwhite/c"
import "bitbucket.org/bigwhite/d/v2"

func main() {
    c.CallC()
    d.CallD()
} 
```

清理 hello module 的 go.mod，仅保留对包 c 的依赖约束:

```go
// sources/go-module/hello/go.mod
module hello

require (
    bitbucket.org/bigwhite/c v1.3.0
) 
```

重新构建 hello module：

```go
$go build hello.go
go: finding bitbucket.org/bigwhite/c v1.3.0
go: finding bitbucket.org/bigwhite/d v1.2.0
go: downloading bitbucket.org/bigwhite/c v1.3.0
go: downloading bitbucket.org/bigwhite/d v1.2.0
go: finding bitbucket.org/bigwhite/d/v2 v2.0.0
go: downloading bitbucket.org/bigwhite/d/v2 v2.0.0

$cat go.mod
module hello

require (
    bitbucket.org/bigwhite/c v1.3.0 
    bitbucket.org/bigwhite/d/v2 v2.0.0 
)

$./hello
call C: v1.3.0
   --> call D:
    call D: v1.2.0
   --> call D end
call D: v2.0.0 
```

我们看到包 c 依然使用的是 d 的 `v1.2.0` 版本，而 main 中使用的包 d 已经是 `v2.0.0` 版本了。

### 2.3 go module 与 vendor

在最初的 go module 设计中，Russ Cox 是想彻底废除掉 `vendor` 机制的，但在 Go 社区的反馈下，`vendor` 机制得以保留，这也是为了兼容 Go 1.11 之前的版本。

Go module 支持通过下面命令将某个 module 的所有依赖复制一份到 module 根路径下的 `vendor` 目录下:

```shell
$ go mod -vendor
$ ls
go.mod    go.sum  hello.go  vendor/
$ cd vendor
$ ls
bitbucket.org/    modules.txt
$ cat modules.txt
# bitbucket.org/bigwhite/c v1.3.0
bitbucket.org/bigwhite/c
# bitbucket.org/bigwhite/d v1.2.0
bitbucket.org/bigwhite/d
# bitbucket.org/bigwhite/d/v2 v2.0.0
bitbucket.org/bigwhite/d/v2 
```

这样即便在 **module-aware mode** 模式下，我们依然可以只用 vendor 下的包来构建 hello module。比如：我们先删除掉 `$GOPATH/pkg/mod` 目录下的缓存 module (可使用 `go clean -modcache` 命令)，然后执行下面命令：

```shell
$ go build -mode=vendor hello.go
$ ./hello
call C: v1.3.0
   --> call D:
    call D: v1.2.0
   --> call D end
call D: v2.0.0 
```

当然生成的 `vendor` 目录还可以兼容 go 1.11 版本之前的 Go 编译器。不过由于 go 1.11 之前的 Go 编译器不支持在 `GOPATH` 之外使用 `vendor` 机制，我们需要将 hello 目录拷贝到 `$GOPATH/src` 下面才能成功编译它。

### 2.4 go.sum

我们看到执行 go build 后，hello module 的当前目录下还多出了一个 `go.sum` 文件：

```shell
$cat go.sum
bitbucket.org/bigwhite/c v1.3.0 h1:crNI04Bw6lm1yyRjJ+8lJX+3amsxeU72mVQ41kjnESA=
bitbucket.org/bigwhite/c v1.3.0/go.mod h1:6p3lkm60SJ7QP5a4oJyLUxbDJeT+w5x5CShTrekjc7o=
bitbucket.org/bigwhite/d v1.2.0 h1:QQawlmsVZWwIsr0ockPCSJjN1QoKd4W0KEJrINdIzY0=
bitbucket.org/bigwhite/d v1.2.0/go.mod h1:6XJNbysZ+/91fhY6/3TKkMNdV/c0pgaubTQWMigKnlY= 
```

go.sum 记录每个依赖库的版本和对应的内容的校验和 (一个哈希值)。每当增加一个依赖项时，如果 `go.sum` 中没有，则会将该依赖项的版本和内容校验和添加到 `go.sum` 中。go 命令会使用这些校验和与缓存在本地的依赖包副本元信息进行比对校验。

以下面这个 `go.sum` 文件为例：

```shell
$cat go.sum
golang.org/x/text v0.3.0 h1:g61tztE5qeGQ89tm6NTjjM9VPIm088od1l6aSorWRWg=
golang.org/x/text v0.3.0/go.mod h1:NqM8EUOU14njkJ3fqMW+pc6Ldnwhi/IjpwHt7yyuwOQ= 
```

如果我修改了 `$GOPATH/pkg/mod/cache/download/golang.org/x/text/@v/v0.3.0.ziphash` 中的值，那么当我执行下面`verify` 子命令时，我们会得到报错信息：

```go
# go mod verify
golang.org/x/text v0.3.0: zip has been modified (/root/go/pkg/mod/cache/download/golang.org/x/text/@v/v0.3.0.zip)
golang.org/x/text v0.3.0: dir has been modified (/root/go/pkg/mod/golang.org/x/text@v0.3.0) 
```



如果没有 “恶意 " 修改，则 verify 会报成功：

```
# go mod verify
all modules verified 
```

注意：`go.sum` 文件不应被用于理解依赖关系，它只是一个 “元信息数据库”。随着项目依赖的演化变更，`go.sum` 文件中会存储着一个 module 的多个版本信息，即使某个版本已经不再被当前 module 所依赖。

### 2.5 清理 go.mod

在将代码提交 / 推回存储库之前，请运行 `go mod tidy` 以确保 module 文件 (`go.mod`) 是最新且准确的。在本地构建、运行或测试代码将随时影响 Go 对 module 文件中内容的更新。运行 `go mod tidy` 可以确保项目具有所需内容的准确和完整的快照，这对团队中的其他人或 CI/CD 环境大有裨益。

### 2.6 升降级依赖关系

如果对 `go mod init` 初始选择的依赖包版本不甚满意，或是第三方依赖包有更新的版本发布，我们日常开发工作中都会对依赖包的版本进行 “升降级”(upgrade 或 downgrade) 的操作。在 “module-aware mode” 下，由于 `go.mod` 和 `go.sum` 都是由 go 工具链维护和管理的，这里不建议手工去修改 `go.mod` 中 `require` 中包的版本号。我们可以通过 `go get` 命令来实现我们的目的。

我们可以先用 `go list` 命令查看一下某 module 都有哪些版本可用，以 [gocmpp](https://www.helloworld.net/special/jnwqhy/3521160724) 这个项目依赖的 `golang.org/x/text` 为例：

```go
$go list -m -versions golang.org/x/text
golang.org/x/text v0.1.0 v0.2.0 v0.3.0 v0.3.1 v0.3.2 v0.3.3 
```

如果我们选择将 gocmpp 依赖的 `golang.org/x/text` 从 `v0.3.0` 降级到 `v0.1.0`，我们可以在 gocmpp 的项目顶层目录下执行下面命令：

```shell
# go get golang.org/x/text@v0.1.0
go: finding golang.org/x/text v0.1.0
go: downloading golang.org/x/text v0.1.0 
```

降级后，gocmpp 的 `go.mod` 和 `go.sum` 变成了下面这样：

```go
$ cat go.mod
module github.com/bigwhite/gocmpp

require (
    github.com/dvyukov/go-fuzz v0.0.0-20181106053552-383a81f6d048
    golang.org/x/text v0.1.0
)

$ cat go.sum
github.com/dvyukov/go-fuzz v0.0.0-20181106053552-383a81f6d048 h1:3O5zXlWvrRdioniMPz8pW+pGi+BNEFRtVhvj0GnknbQ=
github.com/dvyukov/go-fuzz v0.0.0-20181106053552-383a81f6d048/go.mod h1:11Gm+ccJnvAhCNLlf5+cS9KjtbaD5I5zaZpFMsTHWTw=
golang.org/x/text v0.1.0 h1:LEnmSFmpuy9xPmlp2JeGQQOYbPv3TkQbuGJU3A0HegU=
golang.org/x/text v0.1.0/go.mod h1:NqM8EUOU14njkJ3fqMW+pc6Ldnwhi/IjpwHt7yyuwOQ=
golang.org/x/text v0.3.0 h1:g61tztE5qeGQ89tm6NTjjM9VPIm088od1l6aSorWRWg=
golang.org/x/text v0.3.0/go.mod h1:NqM8EUOU14njkJ3fqMW+pc6Ldnwhi/IjpwHt7yyuwOQ= 
```

我们看到 `go.mod` 中依赖的 `golang.org/x/text` 已经从 v0.3.0 自动变成了 v0.1.0 了。`go.sum` 中也增加了 `golang.org/x/text v0.1.0` 的条目，不过 v0.3.0 的条目依旧存在，我们可以通过 go mod tidy 清理一下：

```go
$ go mod tidy
$ cat go.sum
github.com/dvyukov/go-fuzz v0.0.0-20181106053552-383a81f6d048 h1:3O5zXlWvrRdioniMPz8pW+pGi+BNEFRtVhvj0GnknbQ=
github.com/dvyukov/go-fuzz v0.0.0-20181106053552-383a81f6d048/go.mod h1:11Gm+ccJnvAhCNLlf5+cS9KjtbaD5I5zaZpFMsTHWTw=
golang.org/x/text v0.1.0 h1:LEnmSFmpuy9xPmlp2JeGQQOYbPv3TkQbuGJU3A0HegU=
golang.org/x/text v0.1.0/go.mod h1:NqM8EUOU14njkJ3fqMW+pc6Ldnwhi/IjpwHt7yyuwOQ= 
```

在 “module-aware mode” 下，`go get -u` 会将当前 module 的所有依赖的包版本 (无论直接依赖还是间接依赖) 都升级到最新的兼容版本。比如：我们在 `gocmpp` 项目顶层目录下执行如下命令：

```go
$ go get -u golang.org/x/text
$ cat go.mod
module github.com/bigwhite/gocmpp

require (
    github.com/dvyukov/go-fuzz v0.0.0-20181106053552-383a81f6d048
    golang.org/x/text v0.3.3 //恢复到0.3.3
) 
```

我们看到刚刚降级回 v0.1.0 的依赖项又自动变回 v0.3.3 了（这是目前 text 包的最新版本，注意仅 minor 号和 patch 号变更了）。

如果仅仅要升级 `patch` 号，而不升级 `minor` 号，可以使用 `go get -u=patch A` 。比如：如果 `golang.org/x/text` 有 v0.1.1 版本，那么 `go get -u=patch golang.org/x/text` 会将 `go.mod` 中的 text 后面的版本号变为 `v0.1.1`，而不是 `v0.3.3`。

处于 `module-aware` 工作模式下的 `go get` 更新某个依赖 (无论是升版本还是降版本) 时，会自动计算并更新其间接依赖的包的版本。下面是 `go get` 的其他一些常见命令行选项或参数的含义：

- -t：包括测试代码所依赖的 module；
- -d：下载每个 module 的源代码，但不要构建或安装它们；
- -v：提供详细输出；
- ./… ：在整个源代码树中执行这些操作，并且仅更新所需的依赖项 (不包括测试代码)。

## 3. go module 代理

### 3.1 GOPROXY 环境变量

Go 1.11 版本在引入 go module 的同时，还引入了 Go module proxy。`go get` 命令默认情况下，无论是在 `gopath mode` 还是 `module-aware mode` 下都是直接从代码托管服务器下载 go module 的，比如：github、gitlab 等。但是 Go 1.11 中，我们可以通过设置 **GOPROXY** 环境变量让 Go 命令从其他 module 代理服务器下载 module。比如：

```go
export GOPROXY=https://goproxy.cn 
```

一旦如上面设置生效，后续 go 命令就会通过 [go module 下载协议](https://golang.org/cmd/go/#hdr-Module_proxy_protocol)与 module 代理交互下载特定版本的 module。有了 module proxy，之前的那些包无法 go get 成功 (比如：`golang.org/x` 下面的包) 或者获取缓慢 (比如：github 有时访问很慢) 的问题就都得到了解决。同时，module proxy 也让 gopher 在 module 和包的获取行为上增加了一层控制和干预能力。

Go 1.13 版本之前，**GOPROXY** 这个环境环境变量的默认值为空，Go 工具链都是直接与类似 `github.com` 这样的代码托管站点通信并获取相关依赖包的数据的；一些第三方 module 代理服务发布后，迁移到 go module 的 gopher 们发现：大多数情况下通过 proxy 获取依赖包数据的速度要远高于直接从代码托管站点获取，因此 GOPROXY 总是会配置上一个值。Go 核心团队也希望 Go 世界能有一个像 `node.js` 那样的中心化的 module 仓库为大家提供服务，于是在 Go 1.13 中将 `https://proxy.golang.org` 作为 `GOPROXY` 环境变量的默认值之一，这也是 Go 提供的官方 module 代理服务。

同时从 Go 1.13 版本开始，`GOPROXY` 环境变量**支持设置为多个 proxy 的列表** (多个 proxy 之间采用逗号分隔)，Go 编译器会按顺序尝试从列表中的 proxy 服务获取依赖包数据，当有 proxy 服务不可达或者是返回的 http 状态码不是 404 也不是 410 时，go 会终止数据获取，否则会尝试向列表中的下一个 proxy 服务获取数据。Go 1.13 中，`GOPROXY` 的默认值为 `https://proxy.golang.org,direct`。当官方代理返回 404 或 410 时，Go 编译器会尝试直接连接依赖 module 的代码托管站点以获取数据。

下面是目前世界各地运行的一些知名 module 代理服务：

- [proxy.golang.org](http://proxy.golang.org/) - Go 官方提供的 module 代理服务；
- [gocenter.io](http://gocenter.io/) - JFrog Artifactory 公司提供的 module 代理服务；
- [mirrors.tencent.com/go](http://mirrors.tencent.com/go) - 腾讯公司提供的 module 代理服务；
- [mirrors.aliyun.com/goproxy](http://mirrors.aliyun.com/goproxy) - 阿里云提供的 module 代理服务；
- [goproxy.cn](http://goproxy.cn/) - 开源 module 代理，由七牛云提供主机运行，是目前中国大陆地区最为稳定的 module 代理服务；
- [goproxy.io](http://goproxy.io/) - 开源 module 代理，有中国 go 社区提供的 module 代理服务；
- Athens - 开源 module 代理，可基于该代理自行搭建 module 代理服务。

### 3.2 GOSUMDB

我们知道 go 会在 go module 启用时在本地建立一个 `go.sum` 文件，用来存储依赖包特定版本的加密校验和。同时，Go 维护下载的软件包的缓存，并在下载时计算并记录每个软件包的加密校验和。在正常操作中，go 命令对照这些预先计算的校验和去检查某仓库下的 `go.sum` 文件，而不是在每次命令调用时都重新计算它们。

在日常开发中，特定 module 版本的校验和永远不会改变。每次运行或构建时，go 命令都会通过本地的 `go.sum` 去检查其本地缓存副本的校验和是否一致。如果校验和不匹配，则 go 命令将报告安全错误，并拒绝运行构建或运行。

在这种情况下，重要的是找出正确的校验和，确定是 `go.sum` 错误还是下载的代码是错误的。如果 `go.sum` 中尚未包含已下载的 module，并且该模块是公共 module，则 go 命令将查询 Go 校验和数据库以获取正确的校验和数据存入 `go.sum`。如果下载的代码与校验和不匹配，则 go 命令将报告不匹配并退出。

Go 1.13 提供了 `GOSUMDB` 环境变量用于配置 Go 校验和数据库的服务地址（和公钥），其默认值为 `"sum.golang.org"`，这也是 Go 官方提供的校验和数据库服务 (大陆 gopher 可以使用 `sum.golang.google.cn`)。出于安全考虑，建议保持 `GOSUMDB` 开启。但如果因为某些因素无法访问 GOSUMDB 时（包括 `sum.golang.google.cn`），可以通过下面命令将其关闭：

```go
$go env -w GOSUMDB=off 
```

`GOSUMDB` 关闭后，Go 编译器就仅能使用本地的 `go.sum` 进行包的校验和校验了。

### 3.3 获取私有 module

有了 `GOPROXY` 配置的公共 module 代理服务后，公共 module 数据的获取变得十分容易和高效。但是如果依赖的是企业内部代码服务器上的 go module 或公共代码托管站点上的私有库 (private repository)，通过配置了公共 module 代理服务获取显然不会达到预期效果。以我在 `github.com` 上建立的一个私有仓库 `github.com/bigwhite/privatemodule` 为例 (实验环境的 `GOPROXY` 设置为 `https://goproxy.cn,direct`)：

```shell
$go get github.com/bigwhite/privatemodule
go get github.com/bigwhite/privatemodule: module github.com/bigwhite/privatemodule: git ls-remote -q origin in /root/go/pkg/mod/cache/vcs/026323f17e7ba34a4d690bb5ac8e44aef5d9f49a296aaaad917f4cb1318d1259: exit status 128:
    fatal: could not read Username for 'https://github.com': terminal prompts disabled
Confirm the import path was entered correctly.
If this is a private repository, see https://golang.org/doc/faq#git_https for additional information. 
```

在本地没有缓存 `github.com` 用户名 / 密码的情况下，`go get` 会报上述错误。我们可以使用`.netrc` 的方式配置访问 `github.com` 的凭证。我们创建 `~/.netrc`，其内容如下：

```shell
// ~/.netrc
machine github.com 
login bigwhite 
password [personal access tokens] 
```

`github.com` 的 **personal access tokens** 可以在 `https://github.com/settings/tokens` 下自助生成一个。配置好 `~/.netrc`，我们再来获取一下 `privatemodule`：

```shell
$go get github.com/bigwhite/privatemodule
go: downloading github.com/bigwhite/privatemodule v0.0.0-20200917051519-a62573a3b770
go get github.com/bigwhite/privatemodule: github.com/bigwhite/privatemodule@v0.0.0-20200917051519-a62573a3b770: verifying module: github.com/bigwhite/privatemodule@v0.0.0-20200917051519-a62573a3b770: reading https://goproxy.cn/sumdb/sum.golang.org/lookup/github.com/bigwhite/privatemodule@v0.0.0-20200917051519-a62573a3b770: 404 Not Found
    server response:
    not found: github.com/bigwhite/privatemodule@v0.0.0-20200917051519-a62573a3b770: invalid version: git fetch -f origin refs/heads/*:refs/heads/* refs/tags/*:refs/tags/* in /tmp/gopath/pkg/mod/cache/vcs/026323f17e7ba34a4d690bb5ac8e44aef5d9f49a296aaaad917f4cb1318d1259: exit status 128:
        fatal: could not read Username for 'https://github.com': terminal prompts disabled 
```

我们看到这次 `go get` 依然没有成功！从输出的错误信息我们知道：go get 依旧通过 `GOPROXY` 去获取 privatemodule，但在使用默认的 `GOSUMDB(sum.golang.org)` 校验 privatemoudle 时报了 404 错误。由于是私有仓库，默认的 `sum.golang.org` 站点自然不会有该仓库的校验信息。

那么对于私有 module，如何让 `go get` 绕过 GOPROXY 呢？Go 1.13 提供了 `GOPRIVATE` 环境变量用于指示哪些仓库下的 module 是私有的，不需要通过 `GOPROXY下`载，也不需要通过 `GOSUMDB` 去验证其校验和。不过要注意的是 `GONOPROXY` 和 `GONOSUMDB` 可以覆盖 `GOPRIVATE` 变量中的设置，因此设置时要谨慎，比如下面的例子：

```shell
GOPRIVATE=pkg.tonybai.com/private
GONOPROXY=none
GONOSUMDB=none 
```

`GOPRIVATE` 指示 `pkg.tonybai.com/private` 下的包无需经过 `GOPROXY` 代理下载，不经过 `GOSUMDB` 验证。但 `GONOPROXY` 和 `GONOSUMDB` 均为 **none**，意味着所有 module，不管是公共的还是私有的，都要经过 `GOPROXY` 下载，经过 `GOSUMDB` 验证。我们可以单独设置 `GOPRIVATE` 来实现 `go get` 不使用 `GOPROXY` 下载我们的 privatemodule 并且无需 `GOSUMDB` 校验：

```
export GOPRIVATE=github.com/bigwhite/privatemodule 
```



我们再次执行 `go get` 命令获取 privatemodule：

```
$go get github.com/bigwhite/privatemodule
go: downloading github.com/bigwhite/privatemodule v0.0.0-20200917051519-a62573a3b770
go: github.com/bigwhite/privatemodule upgrade => v0.0.0-20200917051519-a62573a3b770 
```

这回 privatemodule 被成功下载并缓存到本地。

除了使用 `~/.netrc` 实现配置访问 `github.com` 的凭证信息，我们也可以通过 `ssh` 方式访问 `github` 上的私有仓库。首先就是在 `https://github.com/settings/keys` 页面将你的主机公钥内容 (一般 `~/.ssh/id_rsa.pub`) 添加到 `github.com` 的 `SSH keys` 中；然后在你的 `~/.gitconfig` 中添加下面两行配置：

```shell
// ~/.gitconfig
[url "ssh://git@github.com/"]
	insteadOf = https://github.com/ 
```

## 4. 小结

Go 1.11 引入的 go module 机制是近些年 Go 语言较大的一次变更，同时也基本上解决了 Go 社区多年来对 Go 缺少包依赖管理工具的 “抱怨”。在经历了几个版本的优化和打磨后，go module 已经真正成为了 go 项目**包依赖管理的唯一标准**，强烈建议每个刚刚走入 Go 世界的开发者都要拥抱 go module，**使用 go module 管理包依赖**。

本节要点：

- 了解 Go 包依赖管理的演进历史以及不同方案的问题；
- 掌握 Go module 的定义以及其工作模式；
- 掌握 Go module 的核心思想：语义导入版本 (semantic import versioning) 和最小版本选择 (minimal version selection)；
- 掌握 Go module 的常用操作命令；
- 熟悉 Go module 代理的工作原理以及相关环境变量设置。