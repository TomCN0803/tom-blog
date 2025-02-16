---
title: "「Go系列」扩展并发原语之errgroup.Group的实战与源码解读"
date: 2023-12-17T20:00:00+08:00
draft: false
---

Go 语言是如今高并发服务的首选编程语言之一，其自带的`go`关键字、channel 类型以及标准库`sync`和`sync/atomic`中较为丰富的并发原语，使得 Gopher 们能够很顺手地编写出并发程序。Go 官方也利用了这些特性，维护了一个扩展的并发原语库，这里面提供了功能更丰富、使用更便利、封装性更好的并发原语，该扩展并发库的地址为[golang.org/x/sync](https://pkg.go.dev/golang.org/x/sync "`golang.org/x/sync`")。

本文就将对 Go 官方扩展并发库`golang.org/x/sync`中`errgroup.Group`的使用与实现进行讲解。在阅读本文时，希望你已经对 Go 基础（如 channel）以及标准库并发原语（如`WaitGroup`、`Once`、`Context`）有一定的了解。

## 回顾 sync.WaitGroup

在开始了解`errgroup.Group`之前，我们先回顾一下 Go 标准库`sync`包中`WaitGroup`在并发编程中的使用。

如同其名字一样，`sync.WaitGroup`用于等待一组 goroutine 的执行结束。如下面的一段代码中，我们在`main`函数的主 goroutine 中启动了 4 个 goroutine，使用`sync.WaitGroup`中的`Wait`方法，等待 4 个 goroutine 的执行结束，否则`main`函数执行结束程序会直接退出，并不会等待启动的 4 个 goroutine 执行完成：

```go
package main

import (
 "fmt"
 "sync"
 "time"
)

func main() {
 var wg sync.WaitGroup
 wg.Add(4) // WaitGroup计数加4，启动4个goroutines
 for i := 0; i < 4; i++ {
  go func(i int) {
   defer wg.Done() // WaitGroup计数减1，表示一个goroutine执行完成
   time.Sleep(500 * time.Millisecond)
   fmt.Println("goroutine", i, "work done.")
  }(i)
 }

 wg.Wait() // 阻塞，等待所有goroutine执行完成
}

// OUTPUT：
// goroutine 1 work done.
// goroutine 2 work done.
// goroutine 3 work done.
// goroutine 0 work done.
```

简述`sync.WaitGroup`的原理就是在其内部有着一个计数器，用于记录当前未执行完成的 goroutine 数量，使用者使用封装的`Add`和`Done`方法分别对这个计数器进行加减，当计数器为减为`0`时，唤醒调用`Wait`方法阻塞的 goroutine。

## errgroup.Group 的使用

Go 扩展并发库中`errgroup.Group`原语有着与标准库`sync.WaitGroup`同样是对一组`goroutine`进行管理，且有着相似的 API，但前者提供了功能更强大、使用更便捷的封装：

- 封装`Add`与`Done`方法，无需手动调用；
- 更方便的错误捕获与处理；
- 可使用`Context`取消 goroutine 的执行；
- 限制同一时刻并行执行的 goroutine 数量。

下面来通过一个简单的例子看一下`errgroup.Group`的基础使用：

```go
package main

import (
 "errors"
 "fmt"
 "time"

 "golang.org/x/sync/errgroup"
)

func main() {
 var eg errgroup.Group
 for i := 0; i < 4; i++ {
        // 注意闭包中的变量捕获问题 https://golang.org/doc/faq#closures_and_goroutines
  i := i
  eg.Go(func() error {
   if i == 2 {
    // 其中一个 goroutine 返回错误
    return errors.New("goroutine 2 got an error")
   }
   time.Sleep(500 * time.Millisecond)
   fmt.Println("goroutine", i, "work done.")
   return nil
  })
 }

 if err := eg.Wait(); err != nil {
  fmt.Println(err.Error())
  return
 }
}

// OUTPUT：
// goroutine 3 work done.
// goroutine 1 work done.
// goroutine 0 work done.
// goroutine 2 got an error
```

上述程序与上小节`sync.WaitGroup`例子基本一致，只不过令`i=2`时启动的 goroutine 返回一个错误，以展示`errgroup.Group`的错误处理。`errgroup.Group`可以直接通过`var`声明变量进行使用（称之为零值初始化），使用`Go`方法启动一个 goroutine。在主 goroutine 中调用`Wait`方法等待所有 goroutine 执行完成，不同与`sync.WaitGroup`中的`Wait`方法，`errgroup.Group`的`Wait`会返回一个`error`，其值是所有启动的 goroutine 中返回的第一个错误。这里值得注意的是，例如这段示例代码，即使 goroutine 2 已经返回了错误，`Wait`也会等待其它启动的 goroutine 结束才会结束阻塞，返回`error`。

除了零值初始化，`errgroup`提供了另外一种初始化`errgroup.Group`的函数——`WithContext`，向该函数传入一个`Context`，然后会返回一个`errgroup.Group`实例，以及一个从传入的`Context`派生出来的可以被取消的`Context`（称为`cancelCtx`）：

```go
func WithContext(ctx context.Context) (*Group, context.Context)
```

通过这种方式初始化的`errgroup.Group`可以通过返回的`cancelCtx`对 goroutine 的执行进行控制，且如果某个执行的 goroutine 返回了错误，那么都会调用这个`cancelCtx`对应的`cancel`函数，即触发`cancelCtx`的取消。还是以上面的实例代码为基础，这次我们希望当`i=2`的 goroutine 返回错误后，其它 goroutine 立刻终止并返回：

```go
package main

import (
 "context"
 "errors"
 "fmt"
 "time"

 "golang.org/x/sync/errgroup"
)

func main() {
 eg, ctx := errgroup.WithContext(context.Background())
 for i := 0; i < 4; i++ {
  i := i
  eg.Go(func() error {
   if i == 2 {
    return errors.New("goroutine 2 got an error")
   }
   select {
   case <-time.After(500 * time.Millisecond):
   case <-ctx.Done():
    return ctx.Err()
   }
   fmt.Println("goroutine", i, "work done.")
   return nil
  })
 }

 if err := eg.Wait(); err != nil {
  fmt.Println(err.Error())
  return
 }
}

// OUTPUT：
// goroutine 2 got an error
```

上述代码使用了一个`timer`——`time.After(500 * time.Millisecond)`来模拟了 goroutine 的一次具体工作，使用`select`同时对`timer`和`cancelCtx`进行了监听。由于`i=2`的 goroutine 会立刻返回错误，执行时间肯定远快于`500ms`，因此其它 goroutine 会首先监听到`cancelCtx`事件，并在`timer`结束前返回。

`errgroup.Group`默认可以通过`Go`方法启动无限多个 goroutine，但在处理数据量比较大时，无节制地启用 goroutine 很可能会将主机资源耗尽，因此`errgroup.Group`还提供了`SetLimit(n int)`方法，可以限制当前`errgroup.Group`实例下活跃 goroutine 的数量最大为`n`，当传入值为负数则代表没有限制。若当前活跃的 goroutine 数量已经达到了`n`，此时再调用`Go`方法便会被阻塞住，直到活跃 goroutine 数量降至`n`以下。

然而在实际开发场景中，我们并不是都期望阻塞的，因此配合`SetLimit`方法，`errgroup.Group`同样提供了一个`TryGo`方法：

```go
func (g *Group) TryGo(f func() error) bool
```

`TryGo`会尝试执行传入的`f`函数，其仅会在当前活跃 goroutine 数量符合限制时才会启动 goroutine 去执行`f`，否则直接非阻塞地返回`false`。这里要注意的是，`TryGo`返回的布尔值只是用于代表该 goroutine 是否被成功启动，而执行成功与否是该返回值是无法体现的。

`errgroup.Group`虽然提供了对错误的返回，但是只能返回所有启动的 goroutine 产生的第一个错误，如果不做额外扩展的话后面所产生的错误都会被丢弃，然而在很多时候，获取每个 goroutine 的错误情况又是很必要的。解决思路其实也不复杂，我们可以创建一个长度与所启动 goroutine 数量相同的切片（如果数量固定也可以使用数组，这样性能更好）用于存放每个 goroutine 的错误情况。每个 goroutine 在执行完成后，将其错误值存放到切片对应的索引位置。

下面这段代码中启动了 10 个 goroutine，我们令`i`为偶数时的 goroutine 返回错误：

```go
package main

import (
 "errors"
 "fmt"

 "golang.org/x/sync/errgroup"
)

func main() {
 var g struct {
  eg   errgroup.Group
  errs []error
 }

 const jobCount = 10 // 启动goroutine的数量

 g.errs = make([]error, jobCount) // 用于保存每个goroutine的错误情况
 for i := 0; i < jobCount; i++ {
  i := i
  g.eg.Go(func() (err error) {
   defer func() {
    g.errs[i] = err // 保存每个goroutine的错误情况至errs对应的位置
   }()
   if i%2 == 0 {
    err = fmt.Errorf("goroutine %d got an error", i)
   }
   return
  })
 }

 if err := g.eg.Wait(); err != nil {
  fmt.Println(errors.Join(g.errs...).Error())
 }
}

// OUTPUT:
// goroutine 0 got an error
// goroutine 2 got an error
// goroutine 4 got an error
// goroutine 6 got an error
// goroutine 8 got an error
```

除了对多个错误进行捕获外，同样的思路也可以用于保存 goroutine 所产生的计算结果。

## errgroup.Group 实战

在本节我们将利用`errgroup.Group`完成一个单词数量统计的小项目。需求是实现一个二进制程序（这里起名`wc`，意为 word count），可以读取文本文件，并对其中的单词进行词频统计，并打印在标准输出，例如：

```shell
$ ./wc -f article.txt
a                 9
above             3
abundantly        6
after             3
be                4
bearing           2
beast             4
```

为了实现该需求，我们采用的是 MapReduce 的思想，对输入的文本内容主要进行以下几个步骤的处理：

1. Map 阶段：将输入的文本进行单词分割，将每个映射为“单词——频数”的键值对（表示为`(word: count)`），每个单词初始频次都是`1`；
2. Sort 阶段：将所有`(word: count)`以`word`进行排序；
3. Reduce 阶段：将排序好的`(word: count)`按照`word`进行`count`的合并。

> MapReduce 是一种处理数据的方式，最早是由 Google 公司研究提出的一种面向大规模数据处理的并行计算模型和方法，开源的版本是 [Hadoop](https://github.com/apache/hadoop "Hadoop")。本节项目的实现就是采用了 MapReduce 的思想，实现了一个单机单进程的 MapReduce。

由于篇幅限制，文章中将只展示核心部分代码，完整代码请详见 GitHub 仓库：[wc-example](https://github.com/TomCN0803/wc-example "wc-example")。

我们先来看一下主函数`main`的大致代码：

```go
func main() {
    // 省略了部分非核心代码，详见代码仓库

 f, err := os.Open(inputFile)
 if err != nil {
  _, _ = fmt.Fprintf(os.Stderr, "failed to open file: %s\n", err.Error())
  os.Exit(1)
 }
 defer f.Close()

 eg, ctx := errgroup.WithContext(ctx)
 eg.SetLimit(runtime.GOMAXPROCS(0)) // 设置 goroutine 数量为 CPU 核心数
 input := getInputStream(ctx, eg, f)
 mapped := mapper(ctx, eg, input, mapFn)
 sorted := sorter(ctx, eg, mapped)
 reduced := reducer(ctx, eg, sorted)

 eg.Go(func() error {
  for wc := range reduced {
   if _, err := fmt.Printf("%-15s%4d\n", wc.word, wc.count); err != nil {
    return err
   }
  }
  return nil
 })

 if err := eg.Wait(); err != nil {
  _, _ = fmt.Fprintf(os.Stderr, "failed to process file: %s\n", err.Error())
  os.Exit(1)
 }
}
```

主函数中比较核心的代码就是初始化`errgroup.Group`实例`eg`后，对`getInputStream`、`mapper`、`sorter`和`reducer`函数的调用了。其中`getInputStream`是将输入文本转化为可读 channel，后三者则分别对应了 MapReduce 的三个步骤，它们每个步骤所产生的 channel 都作为下一步操作的输入 channel，`reducer`产生的可读 channel 最终由`eg`启动的另一 goroutine 进行读取与输出。以上正是采用了 Go 官方推荐的并发编程范式之一：[管道模式](https://go.dev/blog/pipelines "管道模式")。

下面我们将分别对这四个关键函数进行展开。

### 读取文件

```go
func getInputStream(ctx context.Context, eg *errgroup.Group, r io.Reader) <-chan string {
 ch := make(chan string)

 eg.Go(func() error {
  defer close(ch)
  sc := bufio.NewScanner(r)
  for sc.Scan() {
   line := sc.Text()
   select {
   case ch <- line:
   case <-ctx.Done():
    return ctx.Err()
   }
  }
  return sc.Err()
 })

 return ch
}
```

`getInputStream`先初始化了一个`chan string`类型的 channel `ch`，随后通过`eg.Go`启动了一个 goroutine 来读取 r 中的数据。该 goroutine 中的闭包函数使用了`bufio.Scanner`进行文件读取，每次读取一整行`line`后就会向`ch`发送，这里需要使用 select 判断下传入的`eg`的上下文`ctx`是否被取消，如果被取消了就直接返回`ctx.Err()`，这种处理在后续其它函数中是很常见的。

### Map 阶段

在 map 阶段，我们需要将输入的文本进行单词分割，并将每个单词映射为“单词——频数”的键值对，即`wordCount`类型：

```go
type wordCount struct {
 word  string
 count int
}
```

```go
var nonAlpha = regexp.MustCompile("[^a-zA-Z]+")

func mapFn(line string) []wordCount {
 var result []wordCount
 for _, w := range strings.Fields(line) {
  // 通过正则替换掉掉非字母字符，并转换成小写
  w = strings.ToLower(nonAlpha.ReplaceAllString(w, ""))
  if w != "" {
   result = append(result, wordCount{word: w, count: 1})
  }
 }
 return result
}

func mapper(ctx context.Context, eg *errgroup.Group, input <-chan string, fn func(string) []wordCount) <-chan wordCount {
 ch := make(chan wordCount)

 eg.Go(func() error {
  defer close(ch)
  for l := range input {
   for _, wc := range fn(l) {
    select {
    case ch <- wc:
    case <-ctx.Done():
     return ctx.Err()
    }
   }
  }
  return nil
 })

 return ch
}
```

`mapper`函数接收文件输入的只读 channel `input`，并返回一个元素类型为`wordCount`的 channel `ch`。`mapper`函数启动了一个 goroutine，其闭包函数中使用了`fn`函数对`input`中的每一行文本进行处理。

为了使得代码清晰，这里将`fn`函数的具体实现与`mapper`解耦。其具体实现是`mapFn`函数，它将每一行文本分割为独立的单词，使用正则表达式将非字母字符替换为空字符串，并将单词统一转换为小写，最后将每个单词映射为`wordCount`类型的键值对。

### Sort 阶段

在`mapper`函数中，我们将每一行文本映射为了多个`wordCount`类型的键值对，但是这些键值对是无序的，因此需要在`sorter`函数中对其进行排序：

```go
func sorter(ctx context.Context, eg *errgroup.Group, input <-chan wordCount) <-chan wordCount {
 ch := make(chan wordCount)

 eg.Go(func() error {
  defer close(ch)
  wcHeap := new(wordCountHeap)
  for wc := range input {
   select {
   case <-ctx.Done():
    return ctx.Err()
   default:
    heap.Push(wcHeap, wc)
   }
  }

  for wcHeap.Len() > 0 {
   select {
   case ch <- heap.Pop(wcHeap).(wordCount):
   case <-ctx.Done():
    return ctx.Err()
   }
  }

  return nil
 })

 return ch
}
```

在这段代码中我们从只读 channel `input`中读取`wordCount`类型的键值对，然后将其放入一个`wordCountHeap`类型的最小堆中，最后从最小堆中取出并发送到返回的 channel `ch`中。我们需要注意这里的同步关系——从堆中读取数据必须发生在向堆中写完所有 map 数据的之后，也就是意味着 reduce 必须发生在 sort 完成之后。

`sorter`排序所使用的`wordCountHeap`类型最小堆实现了标准库`heap`包中的`heap.Interface`接口。为了实现起来方便，我们这里直接基于 slice 实现了一个最小堆：

```go
type wordCountHeap []wordCount

func (w *wordCountHeap) Len() int {
 return len(*w)
}

func (w *wordCountHeap) Less(i int, j int) bool {
 return (*w)[i].word < (*w)[j].word
}
func (w *wordCountHeap) Swap(i int, j int) {
 (*w)[i], (*w)[j] = (*w)[j], (*w)[i]
}

func (w *wordCountHeap) Pop() any {
 v := (*w)[len(*w)-1]
 *w = (*w)[:len(*w)-1]
 return v
}

func (w *wordCountHeap) Push(x any) {
 *w = append(*w, x.(wordCount))
}

```

基于 slice 的最小堆`wordCountHeap`会将数据全部存于内存中，在实际生产中可能会遇到很大的文件，这时候我们就需要考虑其它的堆实现方式了，不过这不是本文的重点。

### Reduce 阶段

Reduce 阶段会将排序后的 wordCount 键值对按照 word 进行 count 的合并，最终得到每个 word 的总 count，reduce 阶段处理后产生的 channel `ch`中的数据关于`word`是有序且唯一的：

```go
func reducer(ctx context.Context, eg *errgroup.Group, input <-chan wordCount) <-chan wordCount {
 ch := make(chan wordCount)

 eg.Go(func() error {
  defer close(ch)
  var wc wordCount
  for in := range input {
   if wc.word != in.word {
    if wc.word != "" {
     select {
     case ch <- wc:
     case <-ctx.Done():
      return ctx.Err()
     }
    }
    wc = in
   }
   wc.count += in.count
  }
  return nil
 })

 return ch
}
```

由于篇幅限制，这里只展示了`wc`的核心代码，主要展示了`errgroup.Group`实现 MapReduce 的核心思路。完整的项目除了文中数据处理的核心代码，还需要考虑 flag 参数解析、日志打印、程序优雅退出等，这些代码都可以在仓库中找到。

## 源码走读

### Group 结构体

首先来看一下`errgroup.Group`这个结构体，字段并不复杂，每个字段的意义已经写在了注释当中：

```go
type Group struct {
 cancel func(error) // cancelCtx的取消函数
 wg sync.WaitGroup  // 等待goroutine执行
 sem chan token     // 基于channel的信号量
 errOnce sync.Once  // 确保err只被赋值一次
 err     error    // 保存启动的goroutine所返回的第一个错误
}
```

我们可以看到，`errgroup.Group`内部封装了`sync.WaitGroup`，用于等待 goroutine 的执行；`err`则用于保存启动的 goroutine 执行所返回的第一个错误，由于多个 goroutine 并发执行，因此使用了`sync.Once`确保`err`只被第一个错误赋值；`sem`是一个基于 channel 实现的信号量，用于控制 goroutine 的并发执行数量，在下文中会有详细的介绍；`cancel`是当通过`WithContext`函数初始化时，才会有的`cancelCtx`的取消函数。

由此可见，如果我们使用`var eg errgroup.Group`这样的零值初始化，会默认使`cancel`和`sem`为空值`nil`，即不使用`cancelCtx`去空值 goroutine 的取消，且 goroutine 并发执行数量也没有限制。

### WithContext

我们再来看下另一种初始化方式的`WithContext`函数源码：

```go
func WithContext(ctx context.Context) (*Group, context.Context) {
 ctx, cancel := withCancelCause(ctx)
 return &Group{cancel: cancel}, ctx
}

//go:build go1.20
func withCancelCause(parent context.Context) (context.Context, func(error)) {
 return context.WithCancelCause(parent) // 该函数是go1.20版本及以上才提供的
}
```

根据传入的父`Context`，调用`withCancelCause`函数生成一个可取消的`cancelCtx`和对应的取消函数`cancel`，赋值给返回的`errgroup.Group`实例。

这个`withCancelCause`函数根据编译代码所使用的 Go 语言版本，其实现是不同的。例如在上面的代码中是 go1.20 版本及以上的`withCancelCause`函数实现，它对[`context.WithCancelCause`](https://pkg.go.dev/context#WithCancelCause "`context.WithCancelCause`")函数进行了封装。`context.WithCancelCause`函数是 go1.20 版本更新时加入到标准库`context`中的，与其前辈`context.WithCancel`函数不同的是，其返回的`cancel`函数参数可以传入一个`error`（`cancel`函数类型为[`context.CancelCauseFunc`](https://pkg.go.dev/context#CancelCauseFunc "`context.CancelCauseFunc`")），而后者的`cacel`函数却无法传参（为[`context.CancelFunc`](https://pkg.go.dev/context#CancelFunc "`context.CancelFunc`")类型）。`context.CancelCauseFunc`类型的取消函数能够传入错误信息，在实际开发中更为实用，因此也推荐大家使用新版本的 Go 进行开发。

这里也给出 go1.20 版本以下的`withCancelCause`函数实现，但细节不再赘述：

```go
//go:build !go1.20
func withCancelCause(parent context.Context) (context.Context, func(error)) {
 ctx, cancel := context.WithCancel(parent)
 return ctx, func(error) { cancel() }
}
```

### Go

下面我们来看最关键的`Go`函数的实现：

```go
func (g *Group) Go(f func() error) {
 if g.sem != nil {
  g.sem <- token{} // 尝试获取一个资源
 }

 g.wg.Add(1)
 go func() {
  defer g.done()

  if err := f(); err != nil {
   g.errOnce.Do(func() {
    g.err = err
    if g.cancel != nil {
     g.cancel(g.err)
    }
   })
  }
 }()
}
```

抛开对信号量`sem`的操作，`Go`函数实际上就是对`sync.WaitGroup`的封装。使用`Add(1)`增加一次计数后，启动一个 goroutine 去执行传入的函数`f`。如果函数`f`的执行返回了错误，那么将在`errOnce.Do`中尝试以下两步处理：

- 对内部的`err`进行赋值。由于`sync.Once`在并发场景下的单例性，只有所有 goroutine 的第一个错误会被赋值给`err`；
- 如果内部`cancel`函数不为空值`nil`，那么就执行`cancel`函数，传入`err`。同样由于`sync.Once`在并发场景下的单例性，只有所有 goroutine 的第一个错误会被作为`cancel`的错误原因。

以上处理完成后，会执行`defer`执行的`g.done`函数：

```go
func (g *Group) done() {
 if g.sem != nil {
  <-g.sem // 释放一个资源
 }
 g.wg.Done()
}
```

`g.done`函数的实现很简单，忽略信号量`sem`的操作，就只是执行了`wg.Done`。

### Wait

下面我们看下用于等待`Go`函数所启动的 goroutine 执行完成的`Wait`函数的实现：

```go
func (g *Group) Wait() error {
 g.wg.Wait()
 if g.cancel != nil {
  g.cancel(g.err)
 }
 return g.err
}
```

同样不复杂，`Wait`的阻塞等待就是依靠`wg.Wait`，当 goroutine 执行完成，`wg.Wait`从阻塞中唤醒，此时如果`cancel`函数不为空，就会执行它。这里要重点注意的是，不论内部的错误`err`是否为`nil`，`cancel`都会执行，并传入`err`，即`cancelCtx`都会被取消掉，因此通过`WithContext`初始化`errgroup.Group`实例时所返回的`cancelCtx`请**务必只用于该实例相关的任务执行**。

### 基于 channel 的信号量

接下来我们集中对`errgroup.Group`内部实现的信号量机制进行讲解。

`sem`为一个`chan token`类型的 channel，其中`token`就是个空的结构体，在 Go 语言中空结构体并不占用实际内存，在我们不关心 channel 值时，将 channel 元素类型设为`struct{}`是个很好的实践：

```go
type token struct{}
```

了解 Go 语言 channel 的朋友们应该知道，channel 总共分为**无缓冲**和**有缓冲**两种类型的 channel，简述其区别：

- 无缓冲 channel：当向 channel 发送数据时，发送者会被阻塞，直到有接收者接收数据；当从 channel 接收数据时，接收者会被阻塞，直到有发送者发送数据；
- 有缓冲 channel：当向 channel 发送数据时，如果 channel 缓冲区未满，发送者不会被阻塞，否则发送者会被阻塞，直到有接收者接收数据；当从有缓冲 channel 接收数据时，如果 channel 缓冲区不为空，接收者不会被阻塞，否则接收者会被阻塞，直到有发送者发送数据。

根据 channel 的特性，当我们使用`make(chan token, n)`初始化一个缓冲区大小为`n`的 channel，如果把缓冲区的槽位看作资源，相当于初始化了一个资源数量为`n`的信号量。此时向 channel 发送数据可以被看作获取资源，而接收数据则被看作释放资源。如果没有接收者，则前`n`次向 channel 发送都不会阻塞，而第`n+1`次发送则会阻塞，实现了资源的限制，直到有 channel 数据被接收。

在`Go`方法中，如果`sem`不为空，首先要向`sem`中发送一个`token{}`，相当于获取一个资源，只有获取资源成功才可以继续进行 goroutine 的启动：

```go
func (g *Group) Go(f func() error) {
 if g.sem != nil {
  g.sem <- token{}
 }
    // 省略后续代码
}
```

与`Go`类似但非阻塞的`TryGo`方法则使用了`select`，当获取资源不成功时（`sem`缓冲区已满无法写入），会走`default`分支直接返回`false`，不会被阻塞：

```go
func (g *Group) TryGo(f func() error) bool {
 if g.sem != nil {
  select {
  case g.sem <- token{}:
  default:
   return false
  }
 }
    // 剩余方法实现与Go方法基本一致，这里不再展示
}
```

与之对应地，在`f`执行结束后的`done()`方法中，会从`sem`中读取一次，相当于释放一个资源。

`errgroup.Group`在其`SetLimit(n)`方法中，进行了基于 channel 的信号量的初始化，将`sem`赋值为一个资源数量为`n`的信号量：

```go
func (g *Group) SetLimit(n int) {
 if n < 0 {
  g.sem = nil
  return
 }
 if len(g.sem) != 0 {
  panic(fmt.Errorf("errgroup: modify limit while %v goroutines in the group are still active", len(g.sem)))
 }
 g.sem = make(chan token, n)
}
```

根据`SetLimit`传入的信号量资源个数`n`的大小，会有以下三种情况：

- `n < 0`：将`sem`置空，相当于取消信号量，解除了 goroutine 并发执行数量限制；
- `n = 0`：`sem`将成为一个无缓冲的 channel，此时会阻塞所有资源请求，因此之后必须使用`TryGo`而非`Go`，否则会发生死锁导致 panic；
- `n > 0`：初始化资源数量为`n`的信号量。

`SetLimit(n)`函数是可以多次执行的，比如我们可以先令`n = 3`，再令`n = 10`。但在执行`SetLimit`时，需要通过`len(g.sem) == 0`确保当前`sem`缓冲区中没有数据，即没有正在执行的 goroutine，否则在`sem`被赋新值后，原`sem`将不会有接收者，最终将导致 goroutine 无法结束执行。根据源码，如果`len(g.sem) != 0`，`SetLimit`函数会直接触发 panic，因此调用`SetLimit`时需要确定当前没有活跃的 goroutine。

## 总结

本文详细介绍了 Go 扩展并发库中`errgroup.Group`原语的基本使用，并基于`errgroup.Group`实现了一个单词词频统计的小项目，感受了`errgroup.Group`编写 Go 并发程序的便利性。同时，本文也对`errgroup.Group`的源码进行了走读，详细介绍了其内部的实现原理，包括`errgroup.Group`的结构体、`WithContext`函数、`Go`函数、`Wait`函数以及基于 channel 的信号量机制。希望本文能够帮助大家更好地理解`errgroup.Group`的使用与原理，以及 Go 语言中的并发编程。
