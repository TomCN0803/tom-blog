---
title: "「Go系列」浅探一下Go语言的类型系统"
date: 2022-07-08T10:30:06+08:00
draft: false
---

Go语言是强类型语言，编译器在编译阶段会给代码中每一种类型生成对应的类型描述信息并写入到二进制可执行文件中，这些类型描述信息就被称为“**类型元数据**”。

Go语言的类型分为**内置类型**和**自定义类型**。

> Go语言的内置类型：
>
> bool int int8 int16 int32 int64 uint uint8 uint16 uint32 uint64 uintptr float32 float64 complex64 complex128 string slice map channel rune(int32) byte(uint8)

通过以下3种类型进行定义的都是自定义类型（这里插一句，Go中内置类型是不能被赋予自定义方法的，同样下面`T3`是自定义interface，也不能作为方法接收者，只有`T1`和`T2`可以被赋予自定义方法）：

```go
type T1 int

type T2 struct {
    name string
}

type T3 interface {
    F() string
}
```

数据类型虽然很多，但是不管是内置类型还是自定义类型，它各自对应的的类型元数据是全局唯一的，这些类型元数据共同构成了Go语言的类型系统。接下来就把类型元数据进行展开，浅探究一下里面都有些什么结构，什么内容。

## 内置类型元数据

其实不管是内置类型还是自定义类型，它们都需要记录类型名称、类型大小、内存对齐边界、是否为自定义数据类型等信息，这些信息是内置类型和自定义类型公用的，所以它们被放到了[runtime._type](https://github.com/golang/go/blob/master/src/runtime/type.go)结构体中，作为每个类型元数据的Header：

```go
type _type struct {
	size       uintptr
	ptrdata    uintptr // size of memory prefix holding all pointers
	hash       uint32
	tflag      tflag
	align      uint8
	fieldAlign uint8
	kind       uint8
	// function for comparing objects of this type
	// (ptr to object A, ptr to object B) -> ==?
	equal func(unsafe.Pointer, unsafe.Pointer) bool
	// gcdata stores the GC type data for the garbage collector.
	// If the KindGCProg bit is set in kind, gcdata is a GC program.
	// Otherwise it is a ptrmask bitmap. See mbitmap.go for details.
	gcdata    *byte
	str       nameOff
	ptrToThis typeOff
}
```

在`_type`之后存储的是各类型额外需要描述的信息，例如slice的类型元数据在`_type`结构体后面，记录着一个 `*_type`，指向其存储的元素的类型元数据（比如`[]string`，它的`elem`就会指向string的类型元数据）：

```go
type slicetype struct {
	typ  _type
	elem *_type
}
```

举例，`[]string`类型：

![stringptr](/images/Golang/go-type-system/stringslice.jpg)

对于指针类型，我们可以看到它在`_type`后面也额外存储了一个`*_type`指向该指针类型所指向类型的类型元数据：

```go
type ptrtype struct {
	typ  _type
	elem *_type
}
```

举例，`*string`类型：

![stringptr](/images/Golang/go-type-system/stringptr.jpg)

在[runtime/type.go](https://github.com/golang/go/blob/master/src/runtime/type.go)内同样还定义了map、struct等类型的元数据，都是采用了类似的结构。

## 自定义类型元数据

如果是自定义类型，`_type`后面会跟一个`uncommontype`结构体：

```go
type uncommontype struct {
	pkgpath nameOff
	mcount  uint16 // number of methods
	xcount  uint16 // number of exported methods
	moff    uint32 // offset from this uncommontype to [mcount]method
	_       uint32 // unused
}
```

其中：

- pkgpath记录类型所在的包路径
- mcount记录该类型关联方法的数量
- xcount记录该类型公有方法的数量
- moff记录了该类型关联方法相对于此uncommontype结构体的偏移量（字节为单位），也就是此uncommontype结构体距离数组`[mcount]method`起始地址的偏移字节量

方法的类型元数据`method`定义如下：

```go
type method struct {
	name nameOff
	mtyp typeOff
	ifn  textOff
	tfn  textOff
}
```

我们举个例子，假如说我们基于字符串切片（`[]slice`）类型自定义一个`myslice`类型，并给这个自定义类型`myslice`赋予两个方法`Len()`和`Cap()`：

```go
type myslice []string

func (ms *myslice) Len() {
    fmt.Println(len(ms))
}

func (ms *myslice) Cap() {
    fmt.Println(cap(ms))
}
```

那么在`myslice`的类型元数据中，首先要有一个`slicetype`结构体，毕竟`myslice`的基础类型是字符串切片，`slicetype`中的`elem`指向一个`string`类型的元数据；其次因为`myslice`是自定义类型，所以还需要有一个`uncommontype`结构体，该结构体的`mcount=2`，通过`moff`可以定位到`myslice`的方法列表，包含两个方法`Len()`和`Cap()`：

![myslice](/images/Golang/go-type-system/myslice.jpg)

## Type Alias

Go语言中有两个内置类型很特别，`rune`和`byte`，翻一下[相关的Go源码](https://github.com/golang/go/blob/master/src/builtin/builtin.go)可以看到它们俩是这样定义的：

```go
// byte is an alias for uint8 and is equivalent to uint8 in all ways. It is
// used, by convention, to distinguish byte values from 8-bit unsigned
// integer values.
type byte = uint8

// rune is an alias for int32 and is equivalent to int32 in all ways. It is
// used, by convention, to distinguish character values from integer values.
type rune = int32
```

可以发现相比我们自定义类型时多了一个等号，这种形式叫做**类型别名**，我们用`byte`和`rune`分别给`uint8`和`int32`起了个别名。看眼下面的`MyType1`和`MyType2`，其中`MyType1`就是int32类型别名，`MyType2`是基于`int32`的自定义类型。`MyType1`会和`int32`关联到同一个类型元数据，因此属于同一种类型；`MyType2`则会拥有自己的类型元数据，独立于`int32`。

```go
// 类型别名
type MyType1 = int32

// 自定义类型
type MyType2 int32
```

类型别名：

![类型别名](/images/Golang/go-type-system/alias_MyType1.jpg)

自定义类型：

![自定义类型](/images/Golang/go-type-system/custom_MyType2.jpg)

自定义类型

## 结语

通过初步探索Go语言类型系统，我们发现代码中每一种类型都会对应自己的类型元数据，类型定义的方法也可以通过类型元数据找到，那么这对于理解Go的**接口**、**反射**等机制有着很大的帮助。

