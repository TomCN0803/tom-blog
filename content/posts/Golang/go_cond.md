---
title: "「Go系列」被遗忘的并发原语——sync.Cond"
date: 2024-03-10T10:00:00+08:00
draft: false
---

有其它语言并发编程经验的 Gopher 们一定不会对条件变量（Condition Variable）相关的并发原语感到陌生，例如 Java 的`java.utils.concurrent.locks.Condition`、C++的`std::condition_variable`、Python 的`threading.Condition`等。条件变量在并发编程中用于使一个或多个线程（协程）阻塞地等待一个目标条件被满足，当条件被改变时，可以唤醒一个或多个被阻塞的线程（协程）。

与上述语言类似，在 Go 语言标准库中同样提供了条件变量的并发原语——`sync.Cond`，用于阻塞一个或多个等待某个目标条件成立的 goroutine，直到该条件被改变时这一个或多个 goroutine 才被唤醒。然而在实际开发中，`sync.Cond`很少会被 Gopher 们使用到，因为它的作用似乎都能被 Go channel 给替代，觉得使用 channel 才是更“地道”的 Go 语言用法。但这种说法真的可靠吗？有没有哪些特定场景，`sync.Cond`是不可替代的呢？

## sync.Cond 的基本使用

使用`sync.Cond`时需要关联一个“锁”，也就是实现`sync.Locker`接口的具体类型的实例（例如`sync.Mutex`、`sync.RWMutex`等），在**检查或改变目标条件时需要对这把锁进行加锁**。在使用`sync.Cond`的初始化方法`sync.NewCond`时，需要传入将锁实例，得到`sync.Cond`实例，后续通过访问该实例的`L`字段就可以访问到关联的`Locker`。以下是`sync.NewCond`的签名：

```go
func NewCond(l Locker) *Cond
```

`sync.Cond`有三个方法`Wait`、`Signal`和`Broadcast`：

```go
func (c *Cond) Wait()
func (c *Cond) Signal()
func (c *Cond) Broadcast()
```

下面对这三个方法分别进行介绍：

- `Wait`：会阻塞调用者所在的 goroutine，直到被`Signal`或`Broadcast`方法唤醒，`Wait`才会返回。需要**重点留意**，`Wait`方法内部会调用`c.L.Unlock`对该`sync.Cond`实例关联的锁进行**解锁**，然后再对当前 goroutine 进行阻塞，当该 goroutine 被唤醒后又会在返回前调用`c.L.Lock`进行**加锁**，因此，`Wait`的调用者**必须要持有`c.L`这把锁**；
- `Signal`：唤醒其中一个阻塞等待该`sync.Cond`实例的 goroutine。该方法调用者**不一定需要持有`c.L`这把锁**；
- `Broadcast`：作用类似于`Signal`，不过是唤醒**全部**阻塞等待该`sync.Cond`实例的 goroutine，当只有一个阻塞等待的 goroutine 时，`Broadcast`与`Signal`是等价的。该方法调用者同样**不一定需要持有`c.L`这把锁**。

下面我们通过两个例子来感受下`sync.Cond`的使用。

第一个例子，我们模拟**一对一**的`sync.Cond`使用场景：

```go
func main() {
	cond := sync.NewCond(new(sync.Mutex))
	ready := false // cond所等待的条件

	go func() {
		fmt.Println("doing some work...")
		time.Sleep(time.Second)
		ready = true // 不一定加锁，因为只有一个goroutine在写ready
		cond.Broadcast()
	}()

	cond.L.Lock() // 检查目标条件时先加锁
	for !ready {
		cond.Wait()
	}
	cond.L.Unlock() // Wait返回并跳出for循环后需要解锁
	fmt.Println("got work done signal!")
}

// OUTPUT:
// doing some work...
// got work done signal!
```

上述代码中，我们启动了一个 goroutine 用于模拟一个任务的执行，然后在主 goroutine 中等待该任务执行的结束。例子中声明布尔变量`ready`表示该任务是否执行完成，将`ready == true`作为目标条件。主 goroutine 使用`Wait`方法阻塞等待目标条件`ready == true`成立；任务完成执行后，改变`ready`为`true`，并使用`Broadcast`方法通知被阻塞的主 goroutine（由于只有主 goroutine 在阻塞等待，因此`Broadcast`与`Signal`是等价的）。这里重点留意主 goroutine 使用`sync.Cond`进行`Wait`的方式，尤其是加锁时机——检查目标条件时先加锁，Wait 返回并跳出 for 循环后需要解锁。

下面我们再来看**一对多**的场景：

```go
func main() {
	cond := sync.NewCond(new(sync.Mutex))
	ready := false // cond所等待的条件

 var wg sync.WaitGroup
	wg.Add(1)
	go func() {
		defer wg.Done()
		fmt.Println("doing some work...")
		time.Sleep(time.Second)
		ready = true // 不一定加锁，因为只有一个goroutine在写ready
		cond.Broadcast() // 通知多个被阻塞的goroutine
	}()

	wg.Add(5)
	for range 5 {
		go func() {
			defer wg.Done()
			cond.L.Lock()
			for !ready {
				cond.Wait()
			}
			cond.L.Unlock()
			fmt.Println("got work done signal!")
		}()
	}

	wg.Wait()
}

// OUTPUT:
// doing some work...
// got work done signal!
// got work done signal!
// got work done signal!
// got work done signal!
// got work done signal!
```

上述代码与第一个例子的框架基本类似，只不过启动了 5 个 goroutine 同时等待任务执行完成的通知。

总结两个使用`sync.Cond`的注意事项，以免使用时踩坑：

- 调用`Wait`前必须加锁：在前文介绍`Wait`方法时有提到过`Wait`内部会首先对关联的锁进行解锁，一般锁的实现不允许对没有加锁的锁进行解锁，比如在上文的例子中`sync.Cond`关联了标准库`sync.Mutex`互斥锁，若在调用`Wait`前没有加锁，则会触发 panic：`fatal error: sync: unlock of unlocked mutex`。在 Go 语言圈中有个口诀：“等待毕加索”（`Wait`必须加锁，谐音梗退钱！），可以帮助我们记住这一注意点。
- `Wait`唤醒后需要检查目标条件：`sync.Cond`本身只是负责阻塞与唤醒一个或多个 goroutine，并不能保证目标条件一定是满足了的，且当前 goroutine 从`Wait`被唤醒到`Wait`返回之间，当前 goroutine 是没有获得锁的，因此条件可能会被改变。综上，官方推荐我们使用 for 循环的框架去调用`Wait`等待目标条件的成立：

```go
c.L.Lock()
for !condition {
    c.Wait()
}
// make use of condition...
c.L.Unlock()
```

## sync.Cond vs channel

Gopher 们看完上一节中的两个例子肯定会觉得`sync.Cond`完全可以被 Go 原生的 channel 类型代替，两个示例场景中我们都可以使用`close`关闭 channel 的方式去通知所有阻塞等待的 goroutine，以一对多的场景为例：

```go
func main() {
	var wg sync.WaitGroup
	ready := make(chan struct{})

	wg.Add(1)
	go func() {
		defer wg.Done()
		fmt.Println("doing some work...")
		time.Sleep(time.Second)
		close(ready)
	}()

	wg.Add(5)
	for range 5 {
		go func() {
			defer wg.Done()
			<-ready
			fmt.Println("got work done signal!")
		}()
	}

	wg.Wait()
}

// OUTPUT:
// doing some work...
// got work done signal!
// got work done signal!
// got work done signal!
// got work done signal!
// got work done signal!
```

对于一对一的场景，也可以不使用`close`，直接读写无缓冲 channel `ready`，实现同步关系：

```go
func main() {
	ready := make(chan struct{})

	go func() {
		fmt.Println("doing some work...")
		time.Sleep(time.Second)
		close(ready)
	}()

	<-ready
	fmt.Println("got work done signal!")
}

// OUTPUT:
// doing some work...
// got work done signal!
```

对于这两种通知被阻塞 goroutine 的场景，的确用 channel 更简洁高效，且更符合 Go 语言的编写习惯。但是对于一对多的例子，如果我们需要多次进行 goroutine 的阻塞与唤醒，channel 就显得捉襟见肘了——因为一个 channel 只能被`close`关闭一次重复`close`一个 channel 会导致 panic。比如在下面的例子中，进行了 3 次 goroutine 的阻塞与唤醒（这里只是展示`sync.Cond`的多次阻塞与唤醒，为了方便理解所以没有加入`ready`目标条件）：

```go
func main() {
	var wg sync.WaitGroup
	cond := sync.NewCond(new(sync.Mutex))

	wg.Add(1)
	go func() {
		defer wg.Done()
		for i := range 3 {
			fmt.Printf("doing work %d...\n", i)
			time.Sleep(time.Second)
			cond.Broadcast()
		}
	}()

	wg.Add(5)
	for range 5 {
		go func() {
			defer wg.Done()
			for i := range 3 {
				cond.L.Lock()
				cond.Wait()
				cond.L.Unlock()
				fmt.Println("got work done signal!", i)
			}
		}()
	}

	wg.Wait()
}

// OUTPUT：
// doing work 0...
// doing work 1...
// got work done signal! 0
// got work done signal! 0
// got work done signal! 0
// got work done signal! 0
// got work done signal! 0
// doing work 2...
// got work done signal! 1
// got work done signal! 1
// got work done signal! 1
// got work done signal! 1
// got work done signal! 1
// got work done signal! 2
// got work done signal! 2
// got work done signal! 2
// got work done signal! 2
// got work done signal! 2
```

总结`sync.Cond`相比 channel 的一大不可替代的点就是：**有多个被阻塞 goroutine 的场景中，`Broadcast`方法可以多次调用，以多次唤醒被阻塞的全部 goroutine**。

但如果在这里我们非要使用 channel 的话也不是不可以，就是要比使用`sync.Cond`繁杂一些。我们需要给每个阻塞的 goroutine 关联一个 channel，用于其阻塞与唤醒。然后单独实现一个`broadcast`函数，用于将元素`v`传给多个 channel：

```go
func broadcast[T any](v T, outs []chan T) {
	for _, out := range outs {
		out <- v
	}
}
```

然后我们使用`broadcast`进行多次阻塞与唤醒：

```go
func main() {
	outs := make([]chan struct{}, 5)
	for i := range outs {
		outs[i] = make(chan struct{})
	}

	var wg sync.WaitGroup

	wg.Add(1)
	go func() {
		defer wg.Done()
		for i := range 3 {
			_, _ = fmt.Printf("doing work %d...\n", i)
			time.Sleep(time.Second)
			broadcast(struct{}{}, outs)
		}
	}()

	wg.Add(5)
	for i := range 5 {
		go func(c <-chan struct{}) {
			defer wg.Done()
			for j := range 3 {
				<-c
				fmt.Println("got work done signal!", j)
			}
		}(outs[i])
	}

	wg.Wait()
}

// OUTPUT：
// doing work 0...
// doing work 1...
// got work done signal! 0
// got work done signal! 0
// got work done signal! 0
// got work done signal! 0
// got work done signal! 0
// doing work 2...
// got work done signal! 1
// got work done signal! 1
// got work done signal! 1
// got work done signal! 1
// got work done signal! 1
// got work done signal! 2
// got work done signal! 2
// got work done signal! 2
// got work done signal! 2
// got work done signal! 2
```

## sync.Cond 源码浅析

我们先来看一下`sync.Cond`结构体的字段：

```go
type Cond struct {
	noCopy noCopy

	// 在检查目标条件或者修改条件时需要持有的锁
	L Locker

	notify  notifyList
	checker copyChecker
}
```

对外暴露的锁`L`在前文中已经解释过了，是在检查目标条件或者修改条件时需要持有的锁；`noCopy`和`checker`是用于检测`sync.Cond`实例是否有被复制的，关于复制检测的详细讲解可以参考之前的文章——[《Golang 代码运行时类型复制检查器 copyChecker 的实现》](https://mp.weixin.qq.com/s/c9tDVoNt3dzI-FY1c6QsfA "Golang 代码运行时类型复制检查器 copyChecker 的实现")；`notify`是一个 goroutine 的阻塞等待队列，其底层是由`runtime.notifyList`实现的。

`sync.Cond`的三个方法实现很简单，因为主要的复杂逻辑已经被 Go 语言运行时的`runtime.notifyList`实现了。由于篇幅的原因这里不对`runtime.notifyList`相关的逻辑进行详细讲解，其源码位于[runtime/sema.go](https://cs.opensource.google/go/go/+/refs/tags/go1.22.0:src/runtime/sema.go "runtime/sema.go")中，**在今后会计划写一篇对其进行详细讲解的文章，敬请期待**！

### Wait

```go
func (c *Cond) Wait() {
	c.checker.check()
	t := runtime_notifyListAdd(&c.notify) // 加入通知列表
	c.L.Unlock()
	runtime_notifyListWait(&c.notify, t) // 决定是否加入阻塞等待队列
	c.L.Lock() // 从阻塞队列唤醒后再次加锁
}
```

可以看到`Wait`会调用`c.L.Unlock`对该`sync.Cond`实例关联的锁进行解锁，因此调用`Wait`前必须加锁；在`Wait`返回前又会调用`c.L.Unlock`对该`sync.Cond`实例关联的锁进行加锁，因此`Wait`返回后还需要解锁，避免出现**死锁**的情况。

忽略复制检查和加锁/解锁的代码，那么`Wait`所做的就是使用`runtime_notifyListAdd`将调用者所在 goroutine 加入通知列表中，但还需要调用`runtime_notifyListWait`才可以真正决定当前 goroutine 是否需要加入到阻塞等待队列中。

由于调用`runtime_notifyListWait`可能会阻塞当前 goroutine，因此在调用该方法前需要释放锁，这样其它 goroutine 才能够获得锁。

### Signal 与 Broadcast

```go
func (c *Cond) Signal() {
	c.checker.check()
	runtime_notifyListNotifyOne(&c.notify) // 通知一个等待的goroutine
}

func (c *Cond) Broadcast() {
	c.checker.check()
	runtime_notifyListNotifyAll(&c.notify) // 通知所有等待的goroutine
}
```

同样忽略复制检查的代码，`Signal`调用`runtime_notifyListNotifyOne`通知一个等待的 goroutine，如果该 goroutine 存在于阻塞等待队列中，那么将其移除队列并唤醒；`Broadcast`调用`runtime_notifyListNotifyAll`通知所有等待的 goroutine，清空并唤醒阻塞等待队列中所有的 goroutine。

## 总结

本文介绍了 Go 语言标准库提供的条件变量并发原语`sync.Cond`的一般使用方法，并对比其与 Go 原生的 channel 在不同场景时的优劣。然后我们浅析了`sync.Cond`的源码实现，有助于我们对`sync.Cond`使用方式的理解。
