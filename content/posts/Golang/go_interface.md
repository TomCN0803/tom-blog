---
title: "「Go系列」interface是如何实现的？"
date: 2022-07-22T08:31:05+08:00
draft: false
---

Go语言提供interface类型作为接口类型，声明interface时候会把其方法定义在一起，任何其他类型只要实现了这些方法就是实现了这个接口。不同于Java对接口的显式实现，Go语言中的类型对接口的实现是隐式的，不会标明该类型具体实现了哪些接口，那么对于强类型的Go语言，是如何动态地记录实现了该接口的具体类型？空接口和非空接口在底层又有什么区别？让我们来一起探索一下！

## 空接口类型

空接口，即`interface{}`，可以接收任意类型的数据，在Go 1.18泛型特性出来之前实践中往往会使用空接口结合类型断言来实现一些泛型编程的特性，因此它只要记录接收的数据是什么类型、具体数据存在哪就足够了。Go语言对于空接口类型有对应的结构体类型`eface`，翻一翻[源码](https://github.com/golang/go/blob/master/src/runtime/runtime2.go)，可以看到`eface`的数据结构如下：

```go
type eface struct {
	_type *_type         // 指向接口的动态类型元数据
	data  unsafe.Pointer // 指向接口的动态值
}
```

举个例子，空接口变量`e`被赋值之前，其中存储的`_type`和`data`都是`nil`：

```go
var e interface{}
```

![空接口赋值之前](/images/Golang/go-interface/eface_before_assignment.jpg)

此时如果我们打开一个文件，即把`*os.File`类型变量`fd`赋值给`e`：

```go
f, _ := os.Open("foo.txt")
e = f
```

那么`e`的结构就会变成如下的样子：

![空接口赋值之后](/images/Golang/go-interface/eface_after_assignment.jpg)

## 非空接口

不同于空接口，非空接口需要记录方法列表，一个变量要想赋值给一个非空接口类型，这个变量的类型必须要实现该接口要求的所有方法才行，[源码中](https://github.com/golang/go/blob/master/src/runtime/runtime2.go)对非空接口`iface`的定义如下：

```go
type iface struct {
    // 记录了接口要求的方法列表以及与data对应的动态类型信息等
	tab  *itab
    
	data unsafe.Pointer // 指向接口的动态值
}
```

`iface.data`和`eface.data`的作用是相同的，那么关键信息都存储在`iface.tab`中：

```go
type itab struct {
	inter *interfacetype
	_type *_type
	hash  uint32 // copy of _type.hash. Used for type switches.
	_     [4]byte
	fun   [1]uintptr // variable sized. fun[0]==0 means _type does not implement inter.
}
```

`itab.inter`是接口的类型元数据，它里面记录了这个接口类型的描述信息，接口要求的方法列表就记录在`interfacetype.mhdr`这里：

```go
type interfacetype struct {
	typ     _type
	pkgpath name
	mhdr    []imethod
}
```

`itab._type`就是接口的动态类型，也就是被赋给接口类型的那个变量的类型元数据；`itab.hash`是从`itab._type`中拷贝过来的，是类型的哈希值，用于快速判断类型是否相等时使用；`itab.fun`记录的是动态类型实现的那些接口要求的方法的地址，是从方法元数据中拷贝来的，为的是快速定位到方法，如果`itab._type`对应的类型没有实现该接口中的方法，那么就有`fun[0]==0`。

## 非空接口的赋值

如果我们声明一个变量`rw`，是`io.ReadWriter`类型，在被赋值之前，`rw`的`data`是`nil`，`tab`也是`nil`：

```go
var rw io.ReadWriter
```

![非空接口赋值之前](/images/Golang/go-interface/iface_before_assignment.jpg)

下面我们把一个`*os.File`类型的变量`f`，赋值给`rw`：

```go
f, _ := os.Open("test.txt")
rw = f
```

此时`rw`的动态值就是`f`，动态类型就是`*os.File`，`itab.fun`就是记录了`*os.File`实现的`Read`和`Write`方法地址：

![非空接口赋值之后](/images/Golang/go-interface/iface_after_assignment.jpg)

接下来我们再声明一个`io.Writer`类型的变量`w`，并把`f`赋值给`w`，此时`w`和`rw`的动态类型和动态值与`rw`相同，但是两者的接口元数据有所不同，要求的方法列表（`iface.inter.mhdr`）也不相同：

```go
var w io.Writer
w = f
```

![非空接口赋值之后2](/images/Golang/go-interface/iface_after_assignment_2.jpg)

## itab缓存

对于`itab`我们还要额外关注一点，既然一个非空接口类型和一个动态类型就可以确定一个`itab`的内容，那这个`itab`结构体自然是可以被接口类型与动态类型均相同的接口变量复用的。实际上Go语言会将用到的`itab`结构体用键值对缓存起来，并以接口类型和动态类型共同组合为键，以`*itab`为对应的的值，构建成一个哈希表，用于存储与查询`itab`信息。

这个哈希表叫作`itabTable`，其类型元数据如下，和Go内置的`map`有所不同，结构更加简单：

```go
type itabTableType struct {
    size    uintptr             // length of entries array. Always a power of 2.
    count   uintptr             // current number of filled entries.
    entries [itabInitSize]*itab // really [size] large
}
```

需要一个`itab`时，会首先去`itabTable`里查找，计算哈希值时会用到接口类型`itab.inter`和动态类型`itab._type`的类型哈希值：

```go
func itabHashFunc(inter *interfacetype, typ *_type) uintptr {
    return uintptr(inter.typ.hash ^ typ.hash)
}
```

如果能查询到对应的`itab`指针，就直接拿来使用。若没有就要再创建，然后添加到`itabTable`中。

