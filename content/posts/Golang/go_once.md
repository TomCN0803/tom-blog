---
title: "「Go系列」如何让Go同步原语sync.Once变得更好用？"
date: 2023-10-03T20:00:00+08:00
draft: false
---

`sync.Once`是 Go 语言在标准库中所提供的一个同步原语，其早在 Go 的首个稳定版本[go1](https://pkg.go.dev/sync@go1#Once "go1")中就存在于标准库`sync`包内了。`sync.Once`所提供的`Do`方法，可以保证传入的函数`f`只会在当前`sync.Once`实例`o`下被执行一次：

```go
func (o *Once) Do(f func())
```

因此，`sync.Once`常用于并发场景下单例模式的实现，比如下面这个读写系统环境变量的代码片段：

```go
var (
	envs map[string]string // 存放系统环境变量的map
	once sync.Once         // 保证copyEnv只执行一次
	rwx  sync.RWMutex      // 读写锁
)

// 拷贝环境变量至envs
func copyEnv() {
	envs = make(map[string]string)
	for _, e := range os.Environ() {
		kv := strings.SplitN(e, "=", 2)
		envs[kv[0]] = kv[1]
	}
}

func GetEnv(key string) (value string, ok bool) {
	once.Do(copyEnv)

	rwx.RLock()
	defer rwx.RUnlock()

	value, ok = envs[key]
	return
}

func SetEnv(key, value string) {
	once.Do(copyEnv)

	rwx.Lock()
	defer rwx.Unlock()

	envs[key] = value
}
```

上述代码中`copyEnv`函数将系统环境变量切片进行键值拆分，并赋值给了 map 类型变量`envs`，完成了`envs`的初始化。在并发调用`GetEnv`和`SetEnv`读写环境变量时首先**要确保`envs`已经被初始化，且只初始化了一次**。因此我们使用`sync.Once`，在`GetEnv`和`SetEnv`中首先调用`once.Do`保证`copyEnv`函数已经被执行，且只执行了一次，然后再进行相应的 map 读写操作。

> 其实上述代码示例就是标准库中`syscall`包对 Unix 环境变量增、删、改、查操作的大致实现思路，在`syscall`包中同样也是使用了`sync.Once`保证了环境变量 map 在并发调用时的单例性。详细代码请参见`syscall/env_unix.go`源文件。除此之外还有很多标准库都使用了`sync.Once`，如`strings.Replacer`、`io.pipe`、`time.Location`等。

## sync.Once 的实现

熟悉原子（atomic）操作的 Gopher 可能会联想到 CAS（Compare And Swap）操作——在`sync.Once`中存储一个`done uint32`字段，`0`表示函数`f`没有执行，`1`则表示已执行，然后就有如下代码：

```go
import "sync/atomic"

func (o *Once) Do(f func()) {
    if atomic.CompareAndSwapUint32(&o.done, 0, 1) {
        f()
    }
}
```

上述基于 CAS 原子操作的实现方式正是 Go 官方所特别提到的错误的`sync.Once`实现方式。我们来回顾下`sync.Once`的要求：

1. 传入的函数`f`只被执行一次；
2. 传入的函数`f`的执行与返回必须发生于任何`Do`调用的返回之前。

上述基于 CAS 原子操作的实现方式确实满足了函数`f`只执行一次的要求，但却忽略了要求 2，在执行`Do`时如果执行 CAS 失败就直接返回而不考虑函数`f`是否执行完成。设想文章开头举的并发读写环境变量的例子中`sync.Once`使用了这种实现，当一个 goroutine 调用`copyEnv`执行时间比较长，另一 goroutine 在调用`GetEnv`和`SetEnv`时便会对空指针 map 进行读写操作，这是会直接触发 panic 的，是很严重的问题。

Go 官方对`sync.Once`的实现可谓是简约但不简单，在寥寥几行代码中包含了若干知识点：

```go
type Once struct {
	done uint32
	m    Mutex
}

func (o *Once) Do(f func()) {
	if atomic.LoadUint32(&o.done) == 0 {
		o.doSlow(f)
	}
}

func (o *Once) doSlow(f func()) {
	o.m.Lock()
	defer o.m.Unlock()
	if o.done == 0 {
		defer atomic.StoreUint32(&o.done, 1)
		f()
	}
}
```

1. `done uint32`：依然是`0`表示函数`f`没有执行，`1`则表示已执行；排在结构体的第一个字段位置是因为并发场景下绝大部分 goroutine 只会使用到`done`，而用不到`m Mutex`，这样方便 CPU 对`done`进行相关指令优化；
2. `m Mutex`：在`sync.Once`结构体中除了`done`字段还有一个`m`字段，是一把`sync.Mutex`互斥锁，可以用这把锁保护`done`字段的访问，以确保并发场景下只有一个 goroutine 能够执行函数`f`；
3. `atomic.LoadUint32(&o.done) == 0`：利用了 Go 原子操作的内存可见性，如果`done`已经被别的 goroutine 修改为`1`，即`f`已经被执行，则对当前 goroutine 是可见的，即该`LoadUint32`一定为`1`，此时直接返回；
4. `doSlow`封装：将处理第一次执行`f`的逻辑封装在`doSlow`方法中，可以使`Do`方法逻辑变简短，方便编译器对`Do`方法进行内联优化，以提高执行性能（`sync`包中很多同步原语都采用了这种编码策略，如`sync.Mutex`）；
5. `doSlow`逻辑：首先要加锁，然后对`done`进行二次检查，如果还是`0`，则执行`f`；`defer atomic.StoreUint32(&o.done, 1)`保证`f`执行返回后再将`done`置`1`。

> 根据[Go 语言内存模型](https://go.dev/ref/mem#atomic "Go语言内存模型")（Memory Model），如果一个`atomic.LoadXXX`操作`A`观察到了另一个`atomic.StoreXXX`操作`B`，那么`B`一定发生于`A`之前（这里更准确的术语是 synchronized before）。

这里再强调一下`doSlow`方法中对`done`进行二次检查的必要性——函数`f`的返回可能发生在当前 goroutine 进入`doSlow`和获取到锁之间，因此在 goroutine 拿到锁后对`done`的值应再进行一次检查。例如下图中，goroutine `g1`和`g2`同时执行`Do`，二者进入`doSlow`同时竞争锁，`g1`率先拿到锁后执行`f`并将`done`置`1`，随后释放锁，`g2`拿到锁后需要再次对`done`检查，否则就重复执行`f`了。在`f`执行后再调用`Do`的`g3`在`LoadUint32`时就直接返回了：

![`doSlow()`方法对`done`字段进行双重检查](https://files.mdnice.com/user/17908/1a4e03eb-f190-4c21-9ba8-4c4c66a0b60d.png)

## 对 sync.Once 的扩展

`sync.Once`是 Go 并发编程的一大利器，但由于其是一个比较基础的同步原语，在很多场景中需要对其进行一些扩展才可以满足具体使用需求。

`Do`方法所传入的函数要求是`func()`类型，既没有入参也没有返回值，当我们需要封装一个带有参数以及需要返回初始化后资源的单例函数时就需要做一些额外处理了。一般惯用的处理办法是声明一个`sync.Once`全局变量，并构建一个`func()`类型的闭包函数，在函数体中捕获传入参数；将要返回的变量声明为全局变量，在闭包中进行初始化操作。举个例子，这里假设我们要实现一个`func NewConn(target string) (net.Conn, error)`的单例初始化函数——根据传入的地址`target`，返回一个初始化后的 TCP 连接，以及错误`err`，按照上述方式则有如下实现：

```go
var (
    once 	sync.Once
    conn    net.Conn
    connErr error
)

func NewConn(target string) (net.Conn, error) {
	once.Do(func() {
		conn, connErr = net.Dial("tcp", target)
	})
	return conn, connErr
}
```

`Do`传入的匿名函数捕获了`NewConn`传入的`target`参数，并完成了`conn`和`connErr`的初始化，在并发调用`NewConn`函数时，只有一个 goroutine 执行了`net.Dial("tcp", target)`，其余 goroutine 相当于直接读取了初始化完成的全局变量`conn`和`connErr`。

以上是大部分 Gopher 对于这种`sync.Once`应用场景的处理方式，能正确地实现需求但有个缺点就是声明了比较多的全局变量，随着业务不断复杂，过多的全局变量更易于导致编码缺陷（如全局变量名与局部变量名冲突而导致全局变量误被修改）。这里给出一种我认为更优秀的实现方式：

```go
var connOnce struct {
	sync.Once
	conn net.Conn
	err  error
} // 注意connOnce不是类型而是变量！

func NewConn(target string) (net.Conn, error) {
	connOnce.Do(func() {
		connOnce.conn, connOnce.err = net.Dial("tcp", target)
	})
	return connOnce.conn, connOnce.err
}
```

上述代码只声明了一个结构体类型的全局变量`connOnce`，其中内嵌了`sync.Once`，以及要被初始化的`conn`和`err`。相比上一种处理方式，虽然底层逻辑都是利用闭包与全局变量，但这种处理方式使用了更少的全局变量，且将`sync.Once`与需要初始化的资源联系在一起，使用方便的同时还具有更清晰的代码可读性。

> 上述的实现方式在 Go 标准库中也有所采用，如`math/big/sqrt.go`中的`threeOnce`变量和`three() *Float`方法

上述对于`sync.Once`的扩展在实际开发中是很常见的，Go 官方显然也是注意到了这一点，为了我们能够更灵活地使用`sync.Once`，终于在 2023 年 8 月发布的[go1.21.0 版本](https://pkg.go.dev/sync#OnceFunc "go1.21.0 版本")中，在`sync`标准库中增加了`OnceFunc`、`OnceValue`和`OnceValues`三个辅助函数：

```go
func OnceFunc(f func()) func()
func OnceValue[T any](f func() T) func() T
func OnceValues[T1, T2 any](f func() (T1, T2)) func() (T1, T2)
```

这三个函数都是可以根据传入的函数`f`，返回一个只会执行`f`一次的函数，区别就是传入的函数`f`返回值不同——`OnceFunc`没有返回值，而`OnceValue`和`OnceValues`则分别可以返回一个和两个返回值，且`OnceValue`和`OnceValues`利用了 go1.18 加入的泛型，使得两函数的返回值类型为`any`任意类型。

还是以上面初始化 TCP 连接的需求为例，有时候我们并不想为了这样简单的初始化而专门封装一个函数和声明若干全局变量，比如下面这段代码中就利用了`OnceValues`函数，使用更简洁的代码完成了 TCP 连接单例初始化：

```go
package main

import (
	"fmt"
	"net"
	"sync"
)

func main() {
	target := "baidu.com:80"

	newConnFn := sync.OnceValues(
		func() (net.Conn, error) {
			fmt.Println("getting new conn...")
			return net.Dial("tcp", target)
		},
	)

	var wg sync.WaitGroup
	for i := 0; i < 100; i++ {
		wg.Add(1)
		go func() {
			conn, err := newConnFn()
			if err != nil {
				panic(err)
			}
			defer conn.Close()
			wg.Done()
		}()
	}
	wg.Wait()
}

// OUTPUT:
// getting new conn...
```

以允许一个返回值的`OnceValue`函数为例，来详细看一下它的实现，`OnceFunc`和`OnceValues`的实现方式和`OnceValue`是基本一致的，只有函数返回值个数的区别，这里就不多赘述：

```go
func OnceValue[T any](f func() T) func() T {
      var (
          once   Once
          valid  bool // f()是否执行成功
          p      any  // f()如果panic，将panic的值赋给p
          result T    // 函数返回的结果
      )
      g := func() {
          defer func() {
              p = recover()
              if !valid {
                  panic(p)
              }
          }()
          result = f()
          valid = true
      }
      return func() T {
          once.Do(g)
          if !valid {
              panic(p)
          }
          return result
      }
}
```

`OnceValue`函数和我们之前看到的实现思想差不多，函数内部定义了几个变量（变量定义详见注释），同时定义了一个闭包函数`g`，这个`g`才是真正传入`once.Do`方法中的函数。在`g`中，执行函数`f`，如果成功则将`g`闭包外变量`valid`置为`true`，并将返回值赋值给`g`闭包外变量`result`；如果`f`的执行中触发了 panic，则会在`g`中的`defer`函数内用`recover`进行捕获，并将 panic 的值赋值给`g`闭包外变量`p`，然后再次触发`panic(p)`，这样之后其它 goroutine 会与第一个执行`g`的 goroutine 触发同样的 panic。

## 魔改 sync.Once

在前面使用`sync.Once`进行 TCP 连接初始化时，`net.Dial`可能会因为网络暂时的不稳定而导致 error，而在上面的实现方案中，如果有一个 goroutine 执行了`net.Dial`且因为网络波动返回 error 的话，那么之后所有的其它 goroutine 都会得到这个 gouroutine 的 error 返回，再也没有机会进行 TCP 连接初始化的尝试了，即使此时网络波动已经恢复。针对这种情况我们需要一个特殊的`Once`——当传入函数返回`error`时允许其它 goroutine 再次调用该函数：

```go
type OnceX struct {
	done uint32
	m    sync.Mutex
}

func (o *OnceX) Do(f func() error) error {
	if atomic.LoadUint32(&o.done) == 0 {
		return o.doSlow(f)
	}
	return nil
}

func (o *OnceX) doSlow(f func() error) error {
	o.m.Lock()
	defer o.m.Unlock()
	if o.done == 0 {
		if err := f(); err != nil {
			return err
		}
		atomic.StoreUint32(&o.done, 1)
	}
	return nil
}
```

上述代码实现了一个魔改后的`Once`，取名为`OnceX`，调用`Do`方法可以返回 error，同时不改变内部的`done`字段，这样其它 goroutine 还有尝试执行函数的机会，只有函数成功被执行，才会改变`done`。这种`Once`的魔改在一些开源项目中有被使用，例如[TiDB K8s operator](https://github.com/pingcap/tidb-operator/blob/0258de63eb4aaff73e158110a2344614b60123a6/tests/e2e/br/utils/once/once.go#L34-L41 "TiDB K8s operator")，用于其测试框架中配置的单例初始化。

目前我们可以放心大胆地通过`Do`方法，使得传入的函数`f`只会成功调用一次，但`sync.Once`并没有直接的接口可以告诉我们`f`是否已经被执行过了，因为`Once`内部的`done`字段是私有的，而有时候我们需要知道`f`是否执行完成，如果完成了再进行具体的业务逻辑操作（尤其是`f`是个很耗时的函数时）。如果要实现这个需求，就必须要想办法获得`sync.Once`中`done`字段。这里对教各位一招，实现该`sync.Once`的扩展需求：

```go
// Once 对内嵌的sync.Once扩展一个Done方法
type Once struct {
	sync.Once
}

func (o *Once) Done() bool {
    // 通过unsafe.Pointer获得done
	return atomic.LoadUint32((*uint32)(unsafe.Pointer(&o.Once))) == 1
}
```

上面的代码中，我们自定义了一个`Once`类型，里面嵌套了标准库的`sync.Once`，这样我们可以像`sync.Once`一样调用`Do`方法，同时增加了`Done`判断`f`是否执行完成。`Done`方法的代码将`Once`里面内嵌的`sync.Once`的地址转换为`uint32`类型的地址（因为`done`字段就是`uint32`类型），这样就可以使用`atomic.LoadUint32`原子地加载内嵌`sync.Once`的`done`字段，判断`f`是否执行完成了。

在下面这段代码中，展示了对扩展的`Done`方法的使用：

```go
func main() {
	var once Once
	var wg sync.WaitGroup

	wg.Add(1)
	go func() {
		defer wg.Done()
		// 模拟一个比较耗时的函数执行
		once.Do(func() {
			fmt.Println("doing some really hard work...")
			time.Sleep(time.Second)
		})
	}()

	wg.Add(10)
	for i := 0; i < 10; i++ {
		go func() {
			defer wg.Done()
			if !once.Done() {
				// 如果没有完成，非阻塞地返回
				fmt.Println("not finished yet, return!")
				return
			}
			fmt.Println("hard work has finished")
		}()
		time.Sleep(200 * time.Millisecond)
	}

	wg.Wait()
}

// OUTPUT:
// not finished yet, return!
// not finished yet, return!
// not finished yet, return!
// not finished yet, return!
// not finished yet, return!
// hard work has finished
// hard work has finished
// hard work has finished
// hard work has finished
// hard work has finished
```

## 总结

本文介绍了 Go 标准库中的并发原语`sync.Once`的基本使用，然后对其源码实现进行了解析。了解了`sync.Once`原理后我们可以对`sync.Once`进行更高级的玩法以及拓展，实现了如返回值处理、初始化失败处理、判断是否执行完成等。
