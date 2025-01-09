---
title: 深入理解 Go 调度模型 GPM
slug: go-inDepth-GPM-model
categories:
  - Golang
tags:
  - Development language
halo:
  site: https://weblog.silentmo.cn
  name: 2e393e57-e721-479e-8f1f-51cb3a7331a8
  publish: true
---
# 深入理解 Go 调度模型 GPM
> Author mogd 2022-04-28
> Update mogd 2022-05-05
> Adage `Be content with what you have; rejoice in the way things are.  When you realize there is nothing lacking, the whole world belongs to you.`

Go 语言中最大的一个特性就是天生支持并发，而这一功能体现的就是其调度模型 GPM，那么在了解 Go 调度模型 GPM 之前，需要先了解一下并发 (concurrency) 与并行 (parallesim) 的区别

## 前言

并发和并行最开始都是操作系统中的概念，表示的是CPU执行多个任务的方式，但这是两个不同的概念

并发：在操作系统中，是指一个时间段中有几个程序都处于已启动运行到运行完毕之间，且这几个程序都是在同一个处理机上运行

并行：当系统有一个以上CPU时，当一个CPU执行一个进程时，另一个CPU可以执行另一个进程，两个进程互不抢占CPU资源，可以同时进行，这种方式我们称之为并行(Parallel)

这里有一个很重要的一个点，并行是需要系统有多个 CPU 才会出现

举个例子：
> 你同时跟多名女生聊天，在这个过程**看似同时**完成的，但是其实你是在不同的聊天之间来回切换的
> 反过来，多名女生同时跟你聊天，多个女生之间可以在同一个**时间点**发信息，之间是互不影响的

并发其实是**一段时间**内宏观上多个程序同时运行，而并行是指**同一时刻**，多个任务真的在同时运行

**工作负载 (Workloads)**

在考虑并发时，有两种类型的工作负载需要知道：
- **CPU-Bound**: 永远不会造成 `Goroutines` 自然地进入和退出等待状态的情况。这是一项不断进行计算的工作。将 Pi 计算到第 N 位的线程将受 CPU 限制
- **IO-Bound**: 一种导致 `Goroutines` 自然进入等待状态的工作负载。包括通过网络请求访问资源，或对操作系统进行系统调用，或等待事件发生；需要读取文件的 `Goroutine` 即是 `IO-Bound`；包括同步事件（互斥体、原子），它们会导致 `Goroutine` 作为此类的一部分等待

> 对于 CPU-Bound 工作负载，需要并行性来利用并发性; 对于 IO-Bound 工作负载，不需要并行性即可使用并发性

**总结**

并发指定是多个事情，在同一个时间段内同时发生了，多个任务之间是互相抢占资源的

并行是指多个事情，在同一个时间点同时发生了，任务之间不互相抢占资源

## 一、调度模型 GPM

G、P、M 是 Go 调度器的三个核心组件，是 Go 语言天然支持高并发的内在动力

Go 调度器的工作是在一个或多个处理器上运行的多个工作操作系统线程上分发可运行的 goroutine。在多线程计算中，调度出现了两种范式：工作共享和工作窃取

- **工作共享**：当一个处理器生成新线程时，它会尝试将其中的一些线程迁移到其他处理器，希望它们被空闲/未充分利用的处理器使用
- **工作窃取**：未充分利用的处理器主动寻找其他处理器的线程并“窃取”一些

### 1.1 什么是 GPM

G 是 `goroutine` 的缩写，保存 `goroutine` 的一些状态信息 (执行的函数指令及参数，G 保存的任务对象，线程上下文切换，现场保护和现场恢复) 以及 CPU 的一些寄存器的值，相当于操作系统中的进程控制块

> 当 `goroutine` 被调离 CPU 时，调度器负责把 CPU 寄存器的值保存在 g 对象的成员变量之中
> 当 `goroutine` 被调度起来运行时，调度器又负责把 g 对象的成员变量所保存的寄存器值恢复到 CPU 的寄存器

M 是 `machine` 的缩写，是一个线程。所有的 M 有线程栈，每一个 M 对应着一条操作系统的物理线程，承载着 G 的队列

> G 要调度到 M 上才能运行，M 必须关联 P 才可以执行 Go 代码，但当处理阻塞或系统调用中时，可以不需要关联 P

P 是 `processor` 的缩写，只是一个抽象的概念，并不是真正的物理 CPU。P 为 M 的执行提供 "上下文"，保存 M 执行 G 时的一些资源
> P/M 需要进行绑定，构成一个执行单元
> P 决定了同时可以并发任务的数量，通过 `GOMAXPROCS` 限制同时执行用户级任务的操作系统线程,可以通过 `runtime.GOMAXPROCS` 进行指定

GPM 三者相互依赖，G 需要在 M 上才能运行，M 依赖 P 提供资源，P 持有待运行的 G；三者的关系图：
![GPM-relational](https://gallery-lsky.silentmo.cn/i_blog/2025/01//GPM-relational.png)

M 会从与之绑定的 P 本地队列获取可运行的 G，也会从 network poller 获取可运行的 G，还会从其他 P 偷取 G

宏观上看一下 GPM 的状态流转

G 状态流转 (省略了一些垃圾回收状态)：
![G-stat-flow](https://gallery-lsky.silentmo.cn/i_blog/2025/01//G-state-flow.png)

P 状态流转：
![P-stat-flow](https://gallery-lsky.silentmo.cn/i_blog/2025/01//P-state-flow.png)
> 通常情况下（在程序运行时不调整 P 的个数），P 只会在上图中的四种状态下进行切换。 当程序刚开始运行进行初始化时，所有的 P 都处于 `_Pgcstop` 状态， 随着 P 的初始化（`runtime.procresize`），会被置于 `_Pidle`
> \
> 当 M 需要运行时，会 `runtime.acquirep` 来使 P 变成 `Prunning` 状态，并通过 `runtime.releasep` 来释放。
> \
> 当 G 执行时需要进入系统调用，P 会被设置为 `_Psyscall`， 如果这个时候被系统监控抢夺（`runtime.retake`），则 P 会被重新修改为 `_Pidle`。
> \
> 如果在程序运行中发生 GC，则 P 会被设置为 `_Pgcstop`， 并在 `runtime.startTheWorld` 时重新调整为 `_Prunning`

M 状态流转：
![M-stat-flow](https://gallery-lsky.silentmo.cn/i_blog/2025/01//M-state-flow.png)
> M 只有自旋和非自旋两种状态。自旋的时候，会努力找工作；找不到的时候会进入非自旋状态，之后会休眠，直到有工作需要处理时，被其他工作线程唤醒，又进入自旋状态

### 1.2 M 工作过程

![M 工作过程](https://gallery-lsky.silentmo.cn/i_blog/2025/01//M-work-action.png)

第一步，从工作线程本地运行队列中寻找 `goroutine`

第二步，从全局运行队列中寻找 `goroutine`。为了保证调度的公平性，每个工作线程每经过 61次调度就需要优先尝试从全局运行队列中找出一个 `goroutine` 来运行，这样才能保证位于全局运行队列中的 `goroutine` 得到调度的机会。全局运行队列是所有工作线程都可以访问的，所以在访问它之前需要加锁

第三部，从其它工作线程的运行队列中偷取 `goroutine`。如果上一步也没有找到需要运行的 `goroutine`，则调用 `findrunnable` 从其他工作线程的运行队列中偷取 `goroutine`，`findrunnable` 函数在偷取之前会再次尝试从全局运行队列和当前线程的本地运行队列中查找需要运行的 `goroutine`

### 1.3 须知的知识

**栈**
- 普通栈：普通栈指的是调度的 `goroutine` 组成的函数栈，是可增长的栈
- 线程栈：线程栈是由需要将 `goroutine` 放置上的 M 组成，实质上 m 由 `goroutine` 生成，线程栈大小固定 (设置了 m 的数量)

**队列**
全局队列：
- 队列中的 G 被所有 M 全局共享，为了保证数据竞争问题，需要加锁
- M 自旋 (本地队列上没有 G) 会偷取其他队列的一半 G，放置在自身的本地队列上，后续也是先从该队列偷取，没有则从其他本地队列偷取
- `sysmon` 再次扫描发现上次任然在运行的 G 时 (sysmon 初始扫描间隔为 10ns，每次扫描间隔翻倍，最高为10ms)，会将其停止与 M 解绑，放置该队列中

本地队列：该队列存储数据资源相同的任务，每个本地队列都会绑定一个 M，指定其完成任务，没有数据竞争，无需加锁处理，处理速度远高于全局队列

**上下文切换**
简单理解为当时的环境即可，环境可以包括当时程序状态以及变量状态

对于代码中某个值说，上下文是指这个值所在的局部(全局)作用域对象。相对于进程而言，上下文就是进程执行时的环境，具体来说就是各个变量和数据，包括所有的寄存器变量、进程打开的文件、内存(堆栈)信息等

**线程清理**
由于每个P都需要绑定一个 M 进行任务执行，所以当清理线程的时候，只需要将 P 释放（解除绑定）（M就没有任务），即可。P 被释放主要由两种情况：

- 主动释放：最典型的例子是，当执行G任务时有系统调用，当发生系统调用时M会处于阻塞状态。调度器会设置一个超时时间，当超时时会将P释放
- 被动释放：如果发生系统调用，有一个专门监控程序，进行扫描当前处于阻塞的P/M组合。当超过系统程序设置的超时时间，会自动将P资源抢走。去执行队列的其它G任务

> 阻塞是正在运行的线程没有运行结束，暂时让出 CPU

### 1.4 抢占式调度

在 `runtime.main` 中会在后台创建一个检测线程 `sysmon`，它会周期性地做 `epoll` 操作，同时还会检测每个 P 是否运行较长时间，抢占就是在 `sysmon` 中实现的

1. 如果检测到某个 P 状态处于 `Psyscall` 超过了一个 `sysmon` 的时间周期(20us)，并且还有其它可运行的任务，则切换 P
2. 如果检测到某个 P 的状态为 `Prunning`，并且它已经运行了超过 10ms，则会将 P 的当前的 G 的`stackguard` 设置为 `StackPreempt`。这个操作其实是相当于加上一个标记，通知这个 G 在合适时机进行调度
3. 目前这里只是尽最大努力送达，但并不保证收到消息的 `goroutine` 一定会执行调度让出运行权

`sysmon` 会进入一个无限循环, 第一轮回休眠 20us, 之后每次休眠时间倍增, 最终每一轮都会休眠 10ms. `sysmon` 中有 `netpool` (获取 fd 事件), `retake`(抢占), `forcegc`(按时间强制执行 gc), `scavenge heap`(释放自由列表中多余的项减少内存占用)等处理

```go
func sysmon() {
    lasttrace := int64(0)
    idle := 0 // how many cycles in succession we had not wokeup somebody
    delay := uint32(0)
    for {
        if idle == 0 { // start with 20us sleep...
            delay = 20
        } else if idle > 50 { // start doubling the sleep after 1ms...
            delay *= 2
        }
        if delay > 10*1000 { // up to 10ms
            delay = 10 * 1000
        }
        usleep(delay)

        ......
    }       
}
```

**抢占条件**
- 如果 P 在系统调用中，且时长已经过一次 sysmon 后，则抢占；调用 `handoffp` 解除 M 和 P 的关联。

- 如果 P 在运行，且时长经过一次 `sysmon` 后，并且时长超过设置的阻塞时长，则抢占；设置标识，标识该函数可以被中止，当调用栈识别到这个标识时，就知道这是抢占触发的, 这时会再检查一遍是否要抢占

### 1.5 goroutine 调度时机

在以下四种情况下，`goroutine` 可能会发生调度，但并不一定会发生

- 使用关键字 `go`：`go` 关键字创建一个新的 `goroutine`，`Go sheduler` 会考虑调度
- GC：由于进行 GC 的 `goroutine` 也需要在 M 上运行，因此肯定会发生调度。当然，`Go scheduler` 还会做很多其他的调度，例如调度不涉及堆访问的 `goroutine` 来运行。GC 不管栈上的内存，只会回收堆上的内存
- 系统调用：当 `goroutine` 进行系统调用时，会阻塞 M，所以它会被调度走，同时一个新的 `goroutine` 会被调度上来
- 内存同步访问：`atomic`，`mutex`，`channel` 操作等会使 `goroutine` 阻塞，因此会被调度走。等条件满足后（例如其他 `goroutine` 解锁了）还会被调度上来继续运行

## 参考
[1] [面试必考的：并发和并行有什么区别](https://cloud.tencent.com/developer/article/1424249)
[2] [深入Golang调度器之GMP模型](https://www.cnblogs.com/sunsky303/p/9705727.html)
[3] [goroutine 调度器](https://www.bookstack.cn/read/qcrao-Go-Questions/goroutine.md)
