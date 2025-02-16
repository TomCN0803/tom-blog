---
title: "「Go系列」系统弹力设计之断路器的实现"
date: 2024-04-02T10:00:00+08:00
draft: false
---

## 什么是断路器？

断路器（circuit-breaker）是软件系统弹力设计中常用的故障保护机制，提高软件系统的容错能力。断路器的工作方式类似于我们生活中所接触到的电路断路器，或者说是电闸上的“保险丝”，当电路出现问题（如短路）时会自动跳闸，保险丝会“熔断”，这时电路断开，可以起到保护电路上电器的作用，否则轻则导致电器烧坏，重则引发火灾，因此在电路的设计中，断路器这种自我保护装置可以说是不可或缺的。

同样，在计算机的场景中，一个服务可能会发生故障，当故障的严重程度达到某一阈值时，断路器会自动打开，以阻止对该服务的进一步调用，并直接返回一个预先设定好的错误响应，进而避免故障的加剧与扩散，提高系统的容错能力。

断路器同时还需要具备自我修复的能力。当故障发生一段时间后，断路器会尝试关闭，重新恢复对服务的调用，如果调用恢复正常，则断路器会关闭，否则继续打开，直到服务恢复正常。这就类似于我们生活中的电闸，当电路故障排除后，可以手动合闸，尝试恢复电路的通电状态。

断路器一般存在三种状态：闭合（closed）、断开（open）与半开（half-open），这与电路中的术语就很类似。断路器处于闭合状态时，服务正常，用户请求正常执行，断路器将不会对请求进行拦截；当客户请求因故障导致失败，且失败次数达到一定阈值时，断路器会自动打开并进入断开状态，此时断路器会拦截所有用户请求，并返回预先设定好的错误响应；断路器不可能永远处于断开状态，在一段时间后，断路器会尝试关闭，进入半开状态，重新尝试处理用户请求。如果请求处理成功的次数或比率达到一定阈值，则认为服务已经恢复正常，断路器会进入闭合状态，否则继续保持断开。

## 使用 Go 语言实现断路器

本文将以 Go 语言开源项目[go-resiliency](https://www.github.com/eapache/go-resiliency "go-resiliency")中的断路器实现为例，介绍使用 Go 语言的断路器代码设计与实现。go-resiliency 项目中的断路器实现位于`breaker`包中，其实现了一个精炼好用的断路器，知名开源 Go 语言 Apache Kafka 客户端[sarama](https://github.com/IBM/sarama "sarama")项目中就依赖了 go-resiliency 中所实现的断路器。

### 断路器的基础使用

go-resiliency 的断路器提供了一个`New`方法用于初始化一个断路器（`breaker.Breaker`类型）实例：

```go
func New(errorThreshold, successThreshold int, timeout time.Duration) *Breaker
```

输入参数分别为：

- `errorThreshold`：错误阈值，当`timeout`时间内连续发生错误的次数达到该阈值时，断路器会从闭合状态转为断开状态；
- `successThreshold`：成功阈值，当处于半开状态的断路器连续成功处理请求的次数达到该阈值时，断路器会重新闭合；
- `timeout`：超时时间，断路器在超时时间内连续发生错误的次数达到`errorThreshold`时，会从闭合状态转为断开状态；同时该超时时间也是断路器断开状态下的持续时间，当到期后，断路器会变为半开状态。

随后我们可以通过 `breaker` 实例的 `Run` 方法来执行需要断路保护的目标代码块。`Run`方法所接收的目标函数类型为`func() error`，即执行完后会返回一个`error`，如果执行成功返回的就是`nil`。下面我们来看一个简单的 demo：

```go
func main() {
    cnt := 0
    work := func() error {
        if cnt++; cnt >= 3 {
            // 模拟从第三次调用开始出错
            return errors.New("work got an error")
        }
        return nil
    }

    b := breaker.New(5, 1, 5*time.Second)
    for range 10 {
        switch err := b.Run(work); err {
        case nil:
            fmt.Println("work success!")
        case breaker.ErrBreakerOpen:
            fmt.Println(err.Error())
        default:
            fmt.Println("got some other error:", err.Error())
        }
    }
}
```

上述代码中，我们定义了一个`work`函数，该函数会模拟从第 3 次调用开始出错。我们初始化了一个断路器`Breaker`类型实例`b`，通过`b`进行 10 次`work`的调用。我们通过`b.Run(work)`将`work`作为断路器保护的代码块执行，当`work`执行成功时，`b.Run`会返回`nil`；前 5 次`work`出错将会返回`work`的错误；第 6 次出错开始，`b.Run`会直接返回预先定义的错误`breaker.ErrBreakerOpen`，表示断路器已经打开：

```go
var ErrBreakerOpen = errors.New("circuit breaker is open")
```

以上的代码运行输出为：

```shell
work success!
work success!
got some other error: work got an error
got some other error: work got an error
got some other error: work got an error
got some other error: work got an error
got some other error: work got an error
circuit breaker is open
circuit breaker is open
circuit breaker is open
```

### 断路器源码走读

我们在本节来看下断路器是如何实现的。在 go-resiliency 中，断路器的结构体类型定义如下：

```go
type Breaker struct {
    errorThreshold, successThreshold int
    timeout                          time.Duration

    lock              sync.Mutex
    state             State
    errors, successes int
    lastError         time.Time
}
```

- `errorThreshold`，`successThreshold`，`timeout`：断路器的错误阈值、成功阈值、超时时间。这与上文提到的`New`方法的传参定义是一模一样的，这三个属性也正是通过`New`方法进行初始化的；

- `lock`：断路器用到的互斥锁，用于在并发场景下保护断路器的内部状态；

- `state`：断路器的状态，是`breaker.State`类型（定义见下文）。有三种取值分别对应上文所说断路器的三种状态：`Closed`（闭合）、`Open`（断开）和`HalfOpen`（半开）；

- `errors`，`successes`：断路器内部记录的一段时间内错误次数和成功次数；

- `lastError`：最后一次发生错误的时间。

断路器的状态`State`的定义如下，底层类型为`uint32`：

```go
type State uint32

const (
    Closed State = iota
    Open
    HalfOpen
)
```

有了上述字段定义，断路器的初始化方法`New`很好理解，实际上就是初始化了`errorThreshold`、`successThreshold`和`timeout`：

```go
func New(errorThreshold, successThreshold int, timeout time.Duration) *Breaker {
    return &Breaker{
        errorThreshold:   errorThreshold,
        successThreshold: successThreshold,
        timeout:          timeout,
    }
}
```

`Breaker`的`Run`方法是断路器的核心方法，用于执行需要断路保护的代码块，逻辑很精简，可以被编译器进行内联优化：

```go
func (b *Breaker) Run(work func() error) error {
    state := b.GetState()

    if state == Open {
        return ErrBreakerOpen
    }

    return b.doWork(state, work)
}
```

`Run`方法首先通过`GetState`方法获取当前断路器的状态，当断路器处于断开状态（`Open`）时，目标服务是不可以被调用的，属于 Fast Path，直接返回`ErrBreakerOpen`错误；如果为其它两种状态，需要稍微复杂一些的处理逻辑，则走 Slow Path，调用`doWork`方法执行目标服务的代码块。

获取断路器状态的`GetState`方法实现如下，使用的是原子`Load`操作以保证并发调用时的安全，读取了`state`字段当前的值：

```go
func (b *Breaker) GetState() State {
    return (State)(atomic.LoadUint32((*uint32)(&b.state)))
}
```

> 这种 Fast Path 与 Slow Path 结合的设计可以简化方法逻辑，易于编译器进行内联优化以提高代码性能。在 Go 标准库中也有很多类似的例子，典型的如`sync.Mutex`的`Lock`方法、`sync.Once`的`Do`方法等。

在`doWork`方法中，目标函数才真正被调用了，代码如下：

```go
func (b *Breaker) doWork(state State, work func() error) error {
    var panicValue interface{}

    // 执行目标函数，得到执行结果，以及捕获panic
    result := func() error {
        defer func() {
            panicValue = recover()
        }()
        return work()
    }()

    if result == nil && panicValue == nil && state == Closed {
        // 正常情况：目标函数执行正常，且断路器处于闭合状态，直接返回nil结果，也不用加锁
        return nil
    }

    // 处理目标函数执行结果以更新断路器内部状态，这段逻辑会加锁
    b.processResult(result, panicValue)

    if panicValue != nil {
        panic(panicValue)
    }

    return result
}
```

`doWork`会在一个匿名函数中执行目标函数`work`，得到目标函数返回的`result`（其实就是个`error`），以及如果目标函数 panic 了，也会通过`recover`捕获到 panic，赋值给`panicValue`。然后判断`result`和`panicValue`若为空，即代表目标函数正常执行，且当前断路器处于闭合状态时，直接返回`nil`结果，也不用去竞争锁。这是属于大多数的正常情况，也就是目标服务正常且断路器也没断开情况下的调用逻辑。

而当不满足上述正常情况时，就会走到`processResult`方法中，这个方法会处理目标函数执行结果以更新断路器内部状态，且在`processResult`方法中会加锁。之后如果`panicValue`不为空，还会主动触发一次 panic。

`processResult`方法的逻辑会稍微复杂些，代码实现如下：

```go
func (b *Breaker) processResult(result error, panicValue interface{}) {
    b.lock.Lock()
    defer b.lock.Unlock()

    if result == nil && panicValue == nil {
        if b.state == HalfOpen {
            b.successes++
            if b.successes == b.successThreshold {
                b.closeBreaker()
            }
        }
    } else {
        if b.errors > 0 {
            expiry := b.lastError.Add(b.timeout)
            if time.Now().After(expiry) {
                b.errors = 0
            }
        }

        switch b.state {
        case Closed:
            b.errors++
            if b.errors == b.errorThreshold {
                b.openBreaker()
            } else {
                b.lastError = time.Now()
            }
        case HalfOpen:
            b.openBreaker()
        }
    }
}
```

`processResult`方法首先会加锁，然后判断`result`和`panicValue`是否都为空，以判断目标函数是否是正常执行的：

- 如果是正常执行的，且当前断路器处于半开状态（`HalfOpen`），则会将`successes`加 1，若此时`successes`达到成功阈值`successThreshold`时，会调用`closeBreaker`方法关闭断路器；

- 如果不是正常执行的，说明有错误产生，则先会看断路器所记录的错误次数`errors`是否大于 0，也就是之前是否已经有过错误发生。如果有过错误发生，会判断当前时间是否已经超过了设定的超时时间`timeout`，如果超时了，则会将`errors`重置为 0。随后根据当前断路器的状态，分别处理闭合状态和半开状态的情况：

  - 当断路器处于闭合状态时，会将`errors`加 1，若此时`errors`已经达到错误阈值`errorThreshold`，会调用`openBreaker`方法断开断路器，否则将当前时间赋值给`lastError`，以记录最后一次发生错误的时间，而先不去断开断路器；
  - 当断路器处于半开状态时，按照断路器的工作特点，会直接调用`openBreaker`方法断开断路器。

我们再来看下`processResult`方法所用来断开和闭合断路器的`openBreaker`和`closeBreaker`方法的实现：

```go
func (b *Breaker) openBreaker() {
    b.changeState(Open)
    go b.timer()
}

func (b *Breaker) closeBreaker() {
    b.changeState(Closed)
}

func (b *Breaker) timer() {
    time.Sleep(b.timeout)

    b.lock.Lock()
    defer b.lock.Unlock()

    b.changeState(HalfOpen)
}

func (b *Breaker) changeState(newState State) {
    b.errors = 0
    b.successes = 0
    atomic.StoreUint32((*uint32)(&b.state), (uint32)(newState))
}
```

`openBreaker`和`closeBreaker`方法分别都使用了`changeState`方法来改变断路器的状态。在`changeState`方法中，会将`errors`和`successes`都重置为 0，然后通过原子`Store`操作将`state`字段的值更新为新的状态`newState`。`openBreaker`会多启动一个执行`timer`方法的协程，在这个协程中，会首先睡眠`timeout`时间，然后再次加锁，调用`changeState`方法将断路器状态更新为半开状态。

以上就是 go-resiliency 实现断路器的几乎全部源码，通过上述源码走读，我们可以看到其断路器已经实现了断路器的基本功能，且是并发安全的。为了方便大家整理思路，这里给大家画了一张基于`go-resiliency`断路器的状态转移图，核心的工作原理基本上都能在这张状态转移图中体现。

![go-resiliency断路器Breaker状态转移图](https://files.mdnice.com/user/17908/68743d55-4493-4392-ad67-6e1de38be557.png)
