---
title: "「Go系列」代码运行时类型复制检查器copyChecker的实现"
date: 2023-09-22T20:13:00+08:00
draft: false
---

今天在翻看 Golang sync 包源码时发现了一个以前从来没有仔细看过的代码实现——代码运行时类型复制检查器`copyCheker`，它的作用是可以在 Go 代码运行时，检测一个类型实例是否是复制的，如果是复制的则会触发`panic`。`copyCheker`目前仅被标准库中`sync.Cond`这个并发同步原语所使用，当一个`sync.Cond`变量在运行时被复制了，使用了复制的`sync.Cond`便会抛出`panic: sync.Cond is copied`，因此该检查器源码也与`sync.Cond`一同位于`sync/cond.go`。

## 源码走读

我们直接来看一下`copyChecker`的源码：

```go
type copyChecker uintptr

func (c *copyChecker) check() {
 if uintptr(*c) != uintptr(unsafe.Pointer(c)) &&
  !atomic.CompareAndSwapUintptr((*uintptr)(c), 0, uintptr(unsafe.Pointer(c))) &&
  uintptr(*c) != uintptr(unsafe.Pointer(c)) {
  panic("sync.Cond is copied")
 }
}
```

`copyChecker`的源码非常简短，只有一个`check`方法用于检测类型复制，但它也并不容易理解。我们可以看到`copyChecker`底层的类型就是`uintptr`，Go 中专门用`uintptr`类型存储地址的值，因此`copyChecker`初始化时的零值就是`0`，例如`sync.Cond`，直接在其内部加入了`copyChecker`，调用`sync.NewCond`初始化时默认为零值：

```go
type Cond struct {
 noCopy noCopy
 L Locker
 notify  notifyList
 checker copyChecker
}

// NewCond returns a new Cond with Locker l.
func NewCond(l Locker) *Cond {
 return &Cond{L: l}
}
```

`sync.Cond`总共有`Wait`、`Signal`和`Broadcast`三个方法，每个方法的源码中首先就会调用`copyChecker`的`check`方法检查自己是否被复制的`sync.Cond`。我们接下来重点看一下`check`方法。

`check`方法中只有一个`if`判断，如果满足这个看起来比较复杂的条件，则会直接触发`panic: sync.Cond is copied`，直接报错`sync.Cond`被复制了。这个`if`判断条件其实可以看它的逆，即不触发`panic`的条件，这样更好理解与描述：

```go
uintptr(*c) == uintptr(unsafe.Pointer(c)) ||
  atomic.CompareAndSwapUintptr((*uintptr)(c), 0, uintptr(unsafe.Pointer(c))) ||
  uintptr(*c) == uintptr(unsafe.Pointer(c))
```

根据`||`可以分为三组条件，任意条件满足，都可以通过`copyChecker`的检查，我们来逐个解析一下：

1. `uintptr(*c) == uintptr(unsafe.Pointer(c))`：不等式左边`uintptr(*c)`表示`copychecker`类型指针`c`所指向地址中存储的值，其实就是该`copychecker`本身的值；不等式右边`uintptr(unsafe.Pointer(c))`将`copychecker`指针`c`转为 Go 语言任意指针类型`unsafe.Pointer`，然后转为`uintptr`，这样就得到了`copychecker`本身的存储地址，因此整个不等式的含义就是**该`copychecker`的值是否等于`copychecker`所在地址，即`copychecker`是否指向自己**；
2. `atomic.CompareAndSwapUintptr((*uintptr)(c), 0, uintptr(unsafe.Pointer(c)))`：对`copyChecker`使用了`atomic`包中 CAS（Compare And Swap）原子操作——如果`copyChecker`值为`0`，则将其更新为`uintptr(unsafe.Pointer(c)))`，即`copyChecker`自己的地址，换句话说就是**让`copyChecker`指向了自己**。如果 CAS 操作成功会返回`true`，否则`false`；
3. `uintptr(*c) == uintptr(unsafe.Pointer(c))`：和条件 1 完全一样，其目的是为了**双重检查**。由于`copyChecker`会被多个 goroutine 并发使用，因此如果 2 中 CAS 操作失败也有可能是别的 goroutine 抢先 CAS 操作成功，这里再进行一次检查**确保`copyChecker`已经指向了自己**。

那么`copyChecker`为什么就能根据以上条件判断检测到复制呢？`copyChecker` `c1`在进行一次`check`后会指向自己，如果此时我复制了一份`c2 = c1`，那么`c2`和`c1`的值相同，都是存储了`c1`所在的地址，而`c2`所在的地址必不可能与`c1`相同，因此当`c2`调用`check`，会发现`c2`既不指向自己，CAS 操作也无论如何不可能成功（`c2`不为`0`），条件 3 进而也一定不会成立，那么自然就会触发`panic`了。

我们来看一个`copyChecker`在检查`sync.Cond`被复制的例子：

```go
package main

import (
 "fmt"
 "sync"
 "time"
)

func main() {
 c := sync.NewCond(new(sync.Mutex)) // 初始化一个*sync.Cond

 go func() {
  c.L.Lock()
  defer c.L.Unlock()
  c.Wait()
  fmt.Println("wait exit")
 }()

 time.Sleep(100 * time.Millisecond)
 c2 := *c // 这里c2复制了c，copyChecker会panic
 c2.Signal()
}

// OUTPUT:
// panic: sync.Cond is copied
//
// goroutine 1 [running]:
// sync.(*copyChecker).check(...)
//   /usr/local/go/src/sync/cond.go:102
// sync.(*Cond).Signal(0x5f5e100?)
//   /usr/local/go/src/sync/cond.go:82 +0xb8
```

上面这段代码中，首先初始化了一个`*sync.Cond`变量`c`，然后开启了一个新协程，在新协程内调用`Wait()`阻塞，此时会触发一次`c`中`copyChecker`的`check`，使得`copyChecker`指向了自己。在主协程中，我们又声明了一个`sync.Cond`变量`c2`，这个`c2`直接复制了`c`所指向的`sync.Cond`结构体的值，因此`sync.Cond`内部的`copyChecker`也被复制，在`c2`调用`Signal()`时，会触发`c2`中`copyChecker`的`check`，根据上文描述的原理，此时就会触发`panic`。

所以我们此时基本可以保证在c2的copychecker进行check检测之前，c中的copychecker已经指向了自己。

### copyChecker vs. noCopy

如果你照着上面`sync.Cond`复制的例子把代码敲一遍，然后执行静态检测`go vet`命令，会发现`c2 := *c`这一行被提示了如下警告：

```shell
# copychecker
./main.go:20:8: assignment copies lock value to c2: sync.Cond contains sync.noCopy
```

其实如果你使用了如 Goland、VSCode（启用 lint）这种集成度高的 Go 编辑器，就会在这一行出现 warning 波浪线，提示上述警告。

![我VSCode中所给出的提示，与vet工具提示的内容一致](https://files.mdnice.com/user/17908/a3ab849e-035f-4ae0-ab3c-a2f4a241bea2.png)

这一警告其实归功于`sync.Cond`中的`noCopy`字段，其类型名也是`noCopy`，与`sync.Cond`和`copyChecker`在同一文件下：

```go
type noCopy struct{}

// Lock is a no-op used by -copylocks checker from `go vet`.
func (*noCopy) Lock()   {}
func (*noCopy) Unlock() {}
```

`noCopy`就是一个空结构体，然后有`Lock`和`UnLock`方法，但都是空方法，它存在的意义是什么呢？在 Go 代码中具有`Lock`和`UnLock`方法的类型，或者说实现了`sync.Locker`接口的类型，如果在使用中被拷贝了，就会触发`go vet`工具的警告。因此如果我们在业务开发中想自己定义一个不希望在使用时被拷贝的类型，就可以让该类型实现`sync.Locker`接口，或者在该类型内部定义实现`sync.Locker`接口的字段。

不同于本文主要讲的`copyChecker`，`noCopy`只是辅助`go vet`在对代码进行静态检测时抛出类型复制的警告，而`copyChecker`则是在运行时发现问题并报出`panic`。
