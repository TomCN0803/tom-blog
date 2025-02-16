---
title: "「Go系列」面试必会！两张图讲清Go语言并发调度策略"
date: 2024-05-20T20:13:00+08:00
draft: false
---

背过面试“八股文”的 Gopher 们肯定了解 Go 语言运行时（Go runtime）的 GMP 模型。Go runtime 时通过 GMP 模型可以完成对 goroutine 的创建、调度、销毁等声明周期的管理，实现 Go 语言优秀的高并发性能。本文将重点关注 GMP 模型的的调度策略，聚焦 Go runtime 的两个问题：

1. Go runtime 什么时候会触发调度？有哪几种情况？
2. Go runtime 调度 goroutine 的算法是什么？

## GMP 模型

进入正篇内容前先对前置知识——GMP 模型进行一个大致回顾。由于 GMP 模型本身并不是本文重点，这里只进行简单的概念梳理。

### G

G 是 goroutine 的抽象，在文中大部分时候 goroutine 就是 G，在 Go 源码中是 g 类型。G 有自己的运行栈、状态、通用寄存器值、要执行任务函数的地址等，G 需要与 P 进行绑定才可以执行。

### M

M 是对操作系统线程的抽象，是拿着 G 所提供的执行上下文，真正去执行的类型，在 Go 源码中是 m 类型。M 同样需要与 P 绑定，由 P 来作为代理去执行 G，因此 M 不需要与 G 进行强绑定，无需保存 G 状态信息，这样 G 也可以实现跨 M 执行。

值得注意的是，每个 M 会保存一个特殊 G 的指针，这个 G 就是 g0：它不用于执行用户函数代码，只用于执行调度策略，是本文内容中的一个重要角色。

### P

P 可以理解为线程本地的调度器，或处理器资源池，在 Go 源码中是 p 类型。P 是 GMP 模型的中枢，对 M 与 G 进行动态有机结合。P 中管理着若干 goroutine 调度相关的资源与上下文，如线程本地 G 队列、本地内存池（mcache）、对象缓存等。

对于 G 而言，P 就可以看作是处理器，只有被 P 调度，G 才能够被执行；对于 M 而言，P 就是其执行 G 的代理，无需 M 自己去关心繁杂调度细节（找到可执行 G、内存资源分配等）。

## 调度策略

Goroutine 的调度无非关注的就是两点：什么时候需要调度，以及如何找到下一个需要执行的 goroutine，这也正好分别对应了文章开头的两个问题。

稍微往底层看下，调度就是一个普通的 G 由于一定的原因需要与当前 M 解绑，而 M 又需要有下一个可以执行的 G。实现这样的调度，就需要用户 G 与 g0 之间进行“反复横跳”。这里我们给出一张宏观的调度图：

![goroutine 调度宏观图](https://files.mdnice.com/user/17908/56dfa1a3-ef13-40a8-bcc3-aa8090991f74.png)

执行用户代码的普通 G 调用 `mcall()` 切换到 g0，g0 通过 `schedule()` 调度的核心方法，找到下一个可执行的 G，更新新旧 G 的状态，将新 G 与 M 进行绑定，最终调用 `gogo()` 方法从 g0 切换到新的 G 去执行。

函数`mcall()` 与 `gogo()` 就是实现用户 G 与 g0 之间“反复横跳”的关键，是对偶的存在关系。

### 调度时机与类型

本小节我们来回答第一个问题：什么时候会触发调度？有哪几种情况？

Goroutine 触发调度的情况可以分为以下几种类型：

1. G 正常结束：这种是最简单的情况，G 在一次调度中就完成了执行任务（也有可能手动执行了 `runtime.GoExit()`），切换到 g0 去执行 `goexit0_m()` 函数将当前 G 置为死亡（`_Gdead`）状态，发起新一轮调度；

2. 主动调度：一种用户 G 主动让渡的方式，当前 G 主动让出 M，这是 goroutine “协程” 概念的体现（严格来说 Goroutine 不是协程，而是绿色线程，协程主打协作，只有主动让渡，而 Goroutine 会进行）。其主要方式是，用户在执行代码中调用了 `runtime.Gosched` 方法，此时当前 G 会当让出执行权，主动进行队列等待下次被调度执行。具体代码可以参考 `runtime/proc.go:GoSched()`，在这个函数中会切换到 g0 去执行 `gosched_m()` 函数；

```go
func Gosched() {
    checkTimeouts()
    mcall(gosched_m)
}
```

3. 被动调度：当 G 因为不满足某种执行条件，G 会陷入阻塞状态（准确说是 `_Gwaiting`），常见的如等待获取互斥锁 `sync.Mutex`、等待 channel 的读写等，此时只有关注的条件达成后 G 才可以被唤醒，重新进入等待执行队列中（变为 `_Grunnable`）。如前面说的阻塞情况，使 G 进入阻塞底层都需要执行 `gopark()` 函数，在这个函数中会切换到 g0 去执行 `park_m()` 函数。与 `gopark()` 函数成对偶关系的 `goready()` 函数可以将阻塞的 G 恢复为待执行状态；

```go
func gopark(unlockf func(*g, unsafe.Pointer) bool, lock unsafe.Pointer, reason waitReason, traceReason traceBlockReason, traceskip int) {
    mp := acquirem()
    gp := mp.curg
    // ...
    mcall(park_m)
}

func goready(gp *g, traceskip int) {
    systemstack(func() {
        ready(gp, traceskip, true)
    })
}
```

4. 抢占调度：这是一种比较特殊的调度方式，因为其机制与其它三者差异较大，因此在上图中也没有体现。前 3 种调度都由当前 M 的 g0 完成，唯独抢占调度不同。因为发起系统调用时需要打破用户态的边界进入内核态，此时 M 也会因系统调用而陷入阻塞，无法主动完成调度的行为。因此，在 Go 进程中会有一个全局监控协程 monitor G 的存在，这个 G 会越过 P 直接与一个 M 进行绑定，不断轮询对所有 P 的执行状况进行监控。倘若发现满足抢占调度的条件，则会以第三方的角色出手干预。

### 找到合适的 G

本小节来回答文章开头的第二个问题：调度 goroutine 的算法是什么？即以什么样的策略找到下一个可执行的 G？

调度的核心方法是位于 `runtime/proc.go` 的 [schedule 函数](https://cs.opensource.google/go/go/+/master:src/runtime/proc.go?q=symbol%3A%5Cbruntime.schedule%5Cb%20case%3Ayes "schedule 函数")，执行该函数的是 g0。`schedule()` 会通过 `findRunnable()` 函数封装的调度算法，找到下一个可执行的 G，然后调用 `execute()`，最终 `execute()` 内部会执行 `gogo()` 函数，执行新的 G。

```go
func schedule() {
    // ...
    gp, inheritTime, tryWakeP := findRunnable() // blocks until work is available
    // ...
    execute(gp, inheritTime)
}
```

下面我们来看一张 `findRunnable()` 函数的流程图，描述了其内部封装的调度算法：

![通过 findRunnable() 找到下一个可执行的 goroutine](https://files.mdnice.com/user/17908/7c6d86fc-9a92-4a3d-917e-93a6ed8a07e2.JPG)

1. 每个 P 中都会有个调度计数器变量 `schedtick`，当每执行 61 次调度后，就会从尝试全局 G 队列中获取一个 G 去执行，这是为了防止全局队列中的 G 饥饿。不过看了下源码发现，除了获取一个 g 用于执行外，还会额外将一个 G 从全局队列转移到当前 P 的本地队列，进一步让全局队列中的 G 也得到更充分的执行机会；

```go
func findRunnable() (gp *g, inheritTime, tryWakeP bool)  {
    // ...
    if pp.schedtick%61 == 0 && sched.runqsize > 0 {
        lock(&sched.lock)
        gp := globrunqget(pp, 1) // 尝试从全局队列获取 G
        unlock(&sched.lock)
        if gp != nil {
            return gp, false, false
        }
    }
    // ...
}
```

2. 尝试从当前 P 本地队列中获取一个可执行的 G，其核心逻辑位于 `runqget()` 方法中。由于 work-stealing 机制的存在，其它 P 可能会访问本地 G 队列去窃取 G，因此即使本地 G 队列逻辑上是 P 独有的，也需要加锁去访问，所以这一行为并不是无锁的（网上很多文章写的有问题），只不过 work-stealing 发生频率远低于从本地队列获取 G，因此获得锁的成功率很高，不需要阻塞；
3. 如果 2 失败，则需要去尝试从全局 G 队列中获取一个 G 去执行。全局 G 队列存储于全局变量 `sched.runq` 中，因此访问需要加锁；
4. 如果 3 失败，则会去看看网络轮询器中有没有就绪的 G，也就是 `netpoll()` 一下，看看返回的就绪 G 的队列是否为空。网络轮询器除了等待 I/O 操作的 G 之外，还有计时器（`time.Timer`、`time.Ticker`）就绪的 G 也会在这里获取到；

```go
func findRunnable() (gp *g, inheritTime, tryWakeP bool)  {
    // ...
    if netpollinited() && netpollAnyWaiters() && sched.lastpoll.Load() != 0 {
        // netpoll(0) 是非阻塞的
        if list, delta := netpoll(0); !list.empty() {
            gp := list.pop()
            casgstatus(gp, _Gwaiting, _Grunnable)
            return gp, false, false
        }
    }
    // ...
}
```

5. 如果 4 失败，则需要触发 work-stealing 机制了，即尝试从其它 P 的本地队列中窃取一部分 G。根据源码可知，work-stealing 每次之多会尝试遍历全局 P 队列 **4 次**，每次遍历随机从一个 P 开始，过程中遇到可以 steal 的 P 就返回，窃取 G 的数量为被窃取的 P 本地队列中**一半**的 G 数量。

```go
func stealWork(now int64) (gp *g, inheritTime bool, rnow, pollUntil int64, newWork bool) {
    pp := getg().m.p.ptr()
    ranTimer := false
    const stealTries = 4
    for i := 0; i < stealTries; i++ {
        stealTimersOrRunNextG := i == stealTries-1
        for enum := stealOrder.start(fastrand()); !enum.done(); enum.next() {
            // ...
        }
    }
    return nil, false, now, pollUntil, ranTime
}
```

## 总结

本文主要总结了 Go 语言的宏观调度策略，使用图文结合的方式，对文章开头提出的两个问题：什么时候需要调度，以及如何找到下一个需要执行的 goroutine 作出了回答。
