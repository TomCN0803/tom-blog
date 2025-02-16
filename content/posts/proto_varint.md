---
title: "带你掌握 Protobuf 中 Varint 原理以及代码实现"
date: 2023-04-12T20:20:05+08:00
draft: false
---

## Varint简介

在我们使用gRPC框架进行服务间通信时，我们需要protobuf来定义服务接口和消息格式。
在进行整数类型数据的传输时，除了定长的`fixed32/64`之外，Protobuf都使用了varint变长编码，甚至包括`bool`和`enum`类型。

Varint编码的应用主要是为了解决传输整数时空间效率的问题，即可以用更小的字节数来表示一个整数以节省空间。
以`uint32`类型为例，每个整数都需要占用4个字节，其最大可以表示的整数为`4294967295`，然而在实际业务场景中，我们很少会用到这么大的整数，例如传输一个芳龄18岁的用户年龄，使用固定大小的`uint32`类型就必须使用4个字节来表示用户年龄，多浪费的3字节的空间并没有承载任何有效的信息。

## Varint编码原理

使用varint编码后的结果，其每一个字节由两部分组成：

1. 最高位（Most Significant Bit, MSB）：标志位，表示后续字节是否还存在有效数据，如果为1，则表示后续字节还存在有效数据，如果为0，则表示后续字节不存在有效数据；
2. 低7位：存储有效数据。

例如，对于整数`1`，其可以用一个字节来进行表示，因此varint编码后的结果为`00000001`，其最高位为0，表示后续字节不存在有效数据。

而对于整数`180`，情况则稍微复杂一些，其编码后的结果为`10110100 00000001`，共占用了2个字节，这个结果是如何得到的呢？

1. 将整数`180`转换为二进制表示：`00000000 10110100`
2. 将二进制表示的整数按照每7位进行分组：`0000001 0110100`
3. 按照低字节在前的方式进行排列：`0110100 0000001`
4. 增加标志位：`10110100 00000001`

![Varint编码过程](https://files.mdnice.com/user/17908/e63a6ecd-7c70-4bcd-b182-c9ec8432aeca.png)

从编码结果得到原始数据的过程则是编码过程的逆过程，即：

1. 读取字节流，遇到最高位为0的字节停止：`10110100 00000001`
2. 去掉每个字节的标志位（最高位）：`0110100 0000001`
3. 按照高字节（相对于原数据）在前的方式进行排列：`0000001 0110100`
4. 拼接后得到原始数据：`00000000 10110100`，即`180`

![Varint解码过程](https://files.mdnice.com/user/17908/56b3c9ee-7590-4458-8e49-464e042a29b2.png)

从以上两个例子可以看出，varint变长编码可以节省空间，提高了整型数据传输的空间效率。
对于`180`仅使用了2个字节进行传输，相比于使用固定大小的`uint32`类型，节省了50%的存储空间。

Protobuf中`uint32`、`uint64`、`int32`、`int64`、`bool`和`enum`类型都直接使用了以上varint的编码方式。
其中`enum`类型会将每个枚举值映射为一个`int32`类型，然后使用varint编码进行传输；`bool`类型会将`true`映射为`1`，`false`映射为`0`，然后同样使用varint编码进行传输。

## Varint编码负数：Zig-Zag编码

以上我们一直没有明确考虑的负整数编码，只讨论了对正整数编码的情况，那么负整数的编码方式是怎样的呢？

学过计算机基础的同学都知道，计算机表示负数时使用的是“补码”。
若使用补码来表示一个负整数`-x`，其计算方式为：`~0 - x + 1`（`~`表示按位取反），相当于对`-x`的绝对值`x`进行取反，然后再加`1`。

例如，对于32位的负整数`-2`，使用补码表示为`11111111 11111111 11111111 11111110`，若直接使用varint编码，则其编码结果为`11111110 11111111 11111111 11111111 00000001`，共占用5个字节。
其实根据补码计算公式可推得，对于任意负整数，其补码的最高位都为1，因此所有负数的补码在进行varint编码时都会占用5个字节，这显然违背了varint编码节省数据传输空间的初衷。

Protobuf中的`int32`、`int64`类型其实就是直接对负整数的补码直接进行了varint编码，因此官方给出的提示是：

> Inefficient for encoding negative numbers — if your field is likely to have negative values, use sint32/sint64 instead.

即如果你的字段可能频繁存在负数，那么请使用`sint32/64`类型，而不是`int32/64`类型。

那么相比`int32`和`int64`类型，为什么官方会更推荐使用`sint32`和`sint64`类型来表示负整数呢？

答案就是`sint32/64`类型使用了zig-zag编码，其编码的方式为：

- 对于正整数`x`：将其映射为`2 * x`（一定是是偶数）
- 对于负整数`-x`：将其映射为`2 * x - 1`（一定是奇数）

正如zig-zag的名字一样（之字形交错），正整数和负整数分别编码为正偶数和正奇数，交错排列。例如`sint32`的zig-zag编码情况如下表所示：

|    原码     |    编码    |
| :---------: | :--------: |
|      0      |     0      |
|     -1      |     1      |
|      1      |     2      |
|     -2      |     3      |
|      2      |     4      |
|     ...     |    ...     |
| 0x7FFFFFFF  | 0xFFFFFFFE |
| -0x80000000 | 0xFFFFFFFF |

Zig-Zag编码后的负整数被转为正整数，然后再使用varint进行编码，这样就可以解决`int32/64`类型中对负数编码空间占用大的问题。

### Zig-Zag编码的代码实现

了解了zig-zag编码的原理之后，我们来用Rust编程语言实现一下。

代码以32位整数为例（64位整数同理），编写Zig-Zag编码函数`zigzag_encode_32`与解码函数`zigzag_decode_32`，直接上代码：

```rust
fn zigzag_encode_32(x: i32) -> u32 {
    ((x << 1) ^ (x >> 31)) as u32
}

fn zigzag_decode_32(x: u32) -> i32 {
    ((x >> 1) as i32) ^ -((x & 1) as i32)
}
```

看到代码可能一脸懵逼，zig-zag的编码和解码分别一行位运算代码就搞定了？我们来分别解释一下这两个位运算。

先来看编码过程，核心的位运算如下：

```rust
(x << 1) ^ (x >> 31)
```

首先，将`x`左移一位，相当于对`x`进行了乘以2，然后将`x`**算数**右移31位，即将`x`最高位移动到最低位，最后将左移的结果与右移的结果进行异或运算。我们根据`x`的正负，分情况讨论一下这番操作后的结果是否符合zig-zag编码要求：

- 当`x >= 0`：`x`左移后变为`2x`，由于最高位是`0`，因此与`0`进行异或得到仍是`2x`不变，最终结果为`2x`，符合zig-zag编码要求；
- 当`x < 0`：`x`左移后变为`2x`，由于最高位是`1`，且有符号整型`i32`的右移是**算数**右移，右移后32位每一位都是`1`，因此与全`1`进行异或相当于对`2x`取反，又由于`2x`为负整数，其取反结果为`2|x| - 1`，符合zig-zag编码要求。

关于`x`为负整数的情况，为什么说“由于`2x`为负整数，其取反结果为`2|x| - 1`”一定成立呢？我们计算`2x`补码时使用公式`~0 - 2|x| + 1`，加括号合并一下就变成了`~0 - (2|x| - 1)`，相当于对`2|x| - 1`直接进行取反，因此就有：

```rust
~(2x) == ~(~(2|x| - 1)) == 2|x| - 1
```

再来看解码过程，核心的位运算如下：

```rust
(x >> 1) ^ -(x & 1)
```

还是分情况讨论，这里我们假设原整数为`n`，编码后的结果为`x`：

- 当`n >= 0`：`x = 2n`，右移一位后变为`n`；由于`x`一定为偶数所以`-(x & 1) = 0`，与`0`进行异或后结果不变，因此最终结果为`n`，符合解码要求；
- 当`n < 0`：`x = 2|n| - 1`，右移一位相当于对`x`进行整除`2`，得到`|n| - 1`；由于`x`一定为奇数所以`-(x & 1) = -1`，与`-1`进行异或相当于与全`1`进行异或，即对`|n| - 1`按位取反，因此最终结果为`n`，符合解码要求。

这里再补充解释一下`n < 0`时`|n| - 1`按位取反为什么等于`n`。
对`|n| - 1`按位取反用式子可以表示为`~0 - (|n| - 1)`，进而有`~0 - |n| + 1`，这正好是`n`的补码表示！

## 不推荐使用varint编码的场景

Varint编码设计虽很精妙，但也有其不适用的场景：

- 大整数：对于数值很大的整数，varint编码可能会比固定长度的编码更消耗空间。例如，官方关于32位整数的建议是当整数大于`2^28`时，使用`fixed32`类型而不是`int32`类型（大于`2^28`时，varint编码需要占用5个字节，而`fixed32`类型只需要4个字节）；对于64位整数，当整数大于`2^56`时，使用`fixed64`类型而不是`int64`类型（大于`2^56`时，varint编码至少需要占用9个字节，而`fixed64`类型只需要8个字节）；

- 需要快速随机访问的数据：由于varint是变长编码，所以我们无法直接通过索引来访问特定的整数，必须从头开始解码，直到找到我们需要的整数，这使得varint编码不适合用于需要快速随机访问的数据。

## Varint Rust代码实现

在本节将示范使用Rust代码，对Varint编码进行实现，并形成一个Rust library crate。这里先使用`cargo new --lib proto-varint`初始化代码根目录，项目起名为`proto-varint`，初始化后的项目结构如下：

```shell
proto-varint
├── Cargo.toml
└── src
    └── lib.rs
```

Varint编码的过程是将整型数据转化成字节序列写入到数据源中（socket、文件等），解码的过程是从数据源中读取字节序列并转化为对应的整型数据。Rust语言对于向数据源的读和写字节分别提供了`std::io::Read`和`std::io::Write`这两个trait，为了方便用户使用，应当对所有实现`std::io::Read` trait的类型实现varint解码功能；对所有实现`std::io::Write` trait的类型实现varint编码功能。

> `std::io::Read`提供了`read`方法，使得每次调用`read`都可以从数据源中读取若干字节至指定的buffer中；`std::io::Write`则提供了`write`方法，使得每次调用`write`都可以将指定buffer中的数据写入数据源中。`write`方法并不一定会将指定buffer中的数据全部成功写入数据源，因此`std::io::Write`也提供了`write_all`方法，其内部可能多次调用`write`，以确保指定buffer中的数据全部成功写入数据源。

这里在`lib.rs`中先写一个框架：

```rust
/// Read varint from source.
pub trait VarintRead: std::io::Read {
    fn read_signed_varint_32(&mut self) -> Result<i32, std::io::Error> {
        todo!("读取int32")
    }
    
    fn read_unsigned_varint_32(&mut self) -> Result<u32, std::io::Error> {
        todo!("读取u32")
    }

    fn read_signed_varint_64(&mut self) -> Result<i64, std::io::Error> {
        todo!("读取i64")
    }
    
    fn read_unsigned_varint_64(&mut self) -> Result<u64, std::io::Error> {
        todo!("读取u64")
    }
}

/// Write varint to source.
pub trait VarintWrite: Write {
    fn write_signed_varint_32(&mut self, value: i32) -> Result<(), std::io::Error> {
    todo!("写入i32")
    }
    
    fn write_unsigned_varint_32(&mut self, value: u32) -> Result<(), std::io::Error> {
        todo!("写入u32")
    }

    fn write_signed_varint_64(&mut self, value: i64) -> Result<(), std::io::Error> {
        todo!("写入i64")
    }

    fn write_signed_varint_64(&mut self, value: u64) -> Result<(), std::io::Error> {
        todo!("写入u64")
    }
}

// Implement VarintRead for any type that implements `std::io::Read`
impl<T: std::io::Read> VarintRead for T {}
// Implement VarintWrite for any type that implements `std::io::Write`
impl<T: std::io::Write> VarintWrite for T {}
```

框架中定义了`VarintRead`和`VarintWrite`两个traits，其中`VarintRead` 为varint解码为整型（读取）的trait，`VarintWrite`为varint编码为字节序列（写入）的trait。`VarintRead`需要实现`read_signed_varint_32/64`（读取32/64位有符号整型）和`read_unsigned_varint_32/64`（读取32/64位无符号整型）的默认方法；`VarintWrite`需要实现`write_signed_varint_32/64`（读取32/64位有符号整型）和`write_unsigned_varint_32/64`（写入32/64位无符号整型）的默认方法。最后两行代码对所有实现`std::io::Read` trait的类型实现`VarintRead` ；对所有实现`std::io::Write` trait的类型实现`VarintWrite` 。

### 实现无符号整型的读写

本节将主要以32位无符号整型的varint读写进行讲述，64位无符号整型的varint读写与其原理一致，仅有一些细节上的差异。

首先来看一下varint写入，即编码的实现：

```rust
fn write_unsigned_varint_32(&mut self, value: u32) -> Result<(), std::io::Error> {
    let mut value = value;
    while value >= 0x80 {
        self.write_all(&[value as u8 | 0x80])?;
        value >>= 7;
    }

    return self.write_all(&[value as u8]);
}
```

下面将对代码中的核心要点进行注释：

1. while循环条件`value >= 0x80`：`0x80`对应的二进制形式为`1000 0000`，即表示`value`的二进制至少需要8位来进行表示，根据前文介绍的varint原理可知这种情况表明后续还需要更多字节对`value`进行存储；
2. `self.write_all(&[value as u8 | 0x80])?`：`value as u8 | 0x80`将value使用一个字节来存储，即保留`value`低8位，并与`0x80`按位或操作，相当于将**最高位设置为`1`，同时也提取出了`value`的最低 7 位**，最终写入到数据源中。这里也使用了Rust的语法糖`?`，如果`write_all`失败则直接返回`std::io::Error`；
3. `value >>= 7`：`value`右移7位，继续处理写入后续有效数据，循环执行1、2、3；
4. `self.write_all(&[value as u8])`：当循环结束，说明此时`value`最多只需要7位进行表示，因此转化为`u8`后直接写入数据源，最高位肯定为`0`。

相对于varint的写入，读取varint的实现过程就略显复杂了：

```rust
const MAX_VARINT_32_BYTES: usize = 5; // Max byte of a varint for 32-bit unsigned integer.

fn read_unsigned_varint_32(&mut self) -> Result<u32, std::io::Error> {
    let mut result = 0;
    let mut shift = 0;
    let mut buf = vec![0u8];
    for i in 0..MAX_VARINT_32_BYTES {
        match self.read(&mut buf)? {
            0 => return Err(std::io::Error::new(std::io::ErrorKind::UnexpectedEof, "EOF")),
            1 => {
                let v = buf[0];
                if v < 0x80 {
                    if i == MAX_VARINT_32_BYTES - 1 && v >= 0x10 {
                        return Err(std::io::Error::new(std::io::ErrorKind::InvalidData, "Overflow"));
                    }
                    return Ok(result | (v as u32) << shift);
                }
                result |= (v as u32 & 0x7f) << shift;
                shift += 7;
            }
            _ => {
                return Err(std::io::Error::new(std::io::ErrorKind::InvalidData, "Invalid data"));
            }
        }
    }
    Err(std::io::Error::new(std::io::ErrorKind::InvalidData, "Overflow"))
}
```

这里使用常量`MAX_VARINT_32_BYTES = 5`表示varint最大会使用5个字节对32位无符号整型进行编码，因此主体的`for`循环也是最多执行`MAX_VARINT_32_BYTES`次，如果超过`MAX_VARINT_32_BYTES`次则表明数据溢出，返回`std::io::ErrorKind::InvalidData`并提示溢出。

在`for`循环开始前，代码初始化了一个长度为1，类型为`u8`的缓冲空间`buf`，每次执行循环时，调用`read`从数据源中读取1字节数据到`buf`中，`read`返回所读取的字节数，这里理论上来讲会正好读取1字节，因此对于0字节说明数据传输不全，返回一个`std::io::ErrorKind::UnexpectedEof`错误，对于其它非1字节的结果返回`std::io::ErrorKind::InvalidData`。

在`for`循环开始前，代码同样初始化了最终结果`result = 0`，和`shift = 0`，即当前读取到的字节`v`中的有效数据需要左移多少位才可以写入`result`。

正确读取1字节后，将该字节`v`与`0x80`进行比较：

- 若`v >= 0x80`：说明`v`最高位为`1`，即后续字节中还有数据需要处理。`result |= (v as u32 & 0x7f) << shift`这行代码首先将`v`与`0x7f`进行按位与，根据varint规则保留了`v`中低7位的有效数据，然后左移`shift`位并与`result`按位或，将`v`中有效数据写入结果的高位中；

- 若`v < 0x80`：说明`v`最高位不为`1`，即后续字节中没有数据需要处理。此时还需要进行数据的溢出检查，如果当前`i == MAX_VARINT_32_BYTES - 1`说明已经读取到`MAX_VARINT_32_BYTES`个字节，在此前提下如果读取到的字节`v >= 0x10`那么最终得到的32位无符号整型数一定会溢出，则返回`std::io::ErrorKind::InvalidData`并提示溢出。这个`0x10`便是32位无符号整数所能够表示的最大值`2^32 - 1`在使用varint进行编码时最高字节的最大值——`0x0F`加1。若通过溢出检查，则将`v`左移`shift`位，与`result`进行按位或，得到最终的结果并返回。这里不要忘记对`shift`进行加7。

以上较详细地对32位无符号整型的varint读写的代码实现进行了讲述，对于64位无符号整型的varint读写来讲实现方式是一模一样的，只存在以下和数据溢出相关的常量的区别：

- `MAX_VARINT_64_BYTES = 10`表示varint最大会使用10个字节对64位无符号整型进行编码，因此主循环也最多执行`MAX_VARINT_64_BYTES`次；
- 对应32位无符号整型的溢出检查时所比较的常量`0x10`，64位无符号整数所能够表示的最大值`2^64 - 1`在使用varint进行编码时最高字节的最大值为`0x01`，因此32位代码中`v >= 0x10`在64位代码中要替换为`v >= 0x02`。

通过varint读取/写入代码实现可以发现在写入时很自然地先对整数的低字节进行处理，在读取时也会先解码出原整型数据的低字节，这也就解释了为什么在前文介绍varint编码解码步骤时为什么要进行高低字节调转了——这样更符合代码实现的习惯。

### 实现有符号整型的读写

前文所介绍的varint编码原理，提到对有符号整型会使用zig-zag编码，将正整数和负整数交替映射到无符号整型空间中。基于现在所实现的无符号整型的varint读写，可以很容易地实现有符号整型的varint读写（这里还是以32位有符号整型为例，64位同理）：

```rust
fn write_signed_varint_32(&mut self, value: i32) -> Result<(), std::io::Error> {
    self.write_unsigned_varint_32(value.zigzag())
}

fn read_signed_varint_32(&mut self) -> Result<i32, std::io::Error> {
    let v = self.read_unsigned_varint_32()?;
    Ok(v.zigzag())
}
```

在`write_signed_varint_32`方法中写入`i32`时，先对`value`进行zig-zag编码，然后将编码结果直接传到无符号整型的varint写方法`write_unsigned_varint_32`；对应`read_signed_varint_32`读方法中，先调用无符号整型varint读方法，并对读出来的无符号整型结果进行zig-zag解码。

### 实现Zig-Zag编码

上一节所展示的有符号整型的varint读写代码需要让`i32`/`i64`类型实现`zigzag`方法将其zig-zag编码为`u32`/`u64`类型，然后需要让`u32`/`u64`类型实现`zigzag`方法将其zig-zag解码为`i32`/`i64`类型，因此需要定义一个`ZigZag`的trait，包含`zigzag`方法：

```rust
pub trait ZigZag<T> {
    fn zigzag(&self) -> T;
}
```

`ZigZag` trait使用了泛型，类型形参`T`为`zigzag`方法的返回类型，此时根据zig-zag编码需求可以写出以下代码，依次使得`i32`/`i64`/`u32`/`u64`实现`ZigZag` trait：

```rust
impl ZigZag<u32> for i32 {
    fn zigzag(&self) -> u32 {
        ((self << 1) ^ (self >> 31)) as u32
    }
}

impl ZigZag<i32> for u32 {
    fn zigzag(&self) -> i32 {
        ((self >> 1) as i32) ^ -((self & 1) as i32)
    }
}

impl ZigZag<u64> for i64 {
    fn zigzag(&self) -> u64 {
        ((self << 1) ^ (self >> 63)) as u64
    }
}

impl ZigZag<i64> for u64 {
    fn zigzag(&self) -> i64 {
        ((self >> 1) as i64) ^ -((self & 1) as i64)
    }
}
```

其中，当`T`为`u32`/`u64`时，`zigzag`需要实现编码；当`T`为`i32`/`i64`时，`zigzag`需要实现解码。Zig-Zag编码解码的代码实现在前文中已有详细解释，这里不再赘述。

目前为止，我们对varint的实现全部写在了`lib.rs`一个源文件中，代码比较臃肿且结构不是很清晰。由于zig-zag的代码逻辑属于比较独立的模块，我们可以将zig-zag的实现独立为一个Rust module。这里我们新建一个`zigzag.rs`源文件，将本节中zig-zag的代码实现复制过去，并且不要忘记在主模块`lib.rs`中添加`mod zigzag`以声明`zigzag`模块（不熟悉Rust module的朋友请参考[Rust官方教程](https://doc.rust-lang.org/stable/book/ch07-00-managing-growing-projects-with-packages-crates-and-modules.html)）。此时我们`proto-varint`项目根目录结构如下：

```shell
proto-varint
├── Cargo.toml
└── src
    ├── lib.rs
    └── zigzag.rs
```

### 使用Rust宏优化重复代码

当前varint的读写代码实现针对32位整数和64位整数分别提供了两套几乎一样的代码，只有一些细节上的不同，例如在varint读时，32位整型的varint最多5字节，而64位最多10字节。因此我们可以使用Rust宏（Macro）写一个通用的模版，减少因32位和64位的不同而导致的重复代码。

我们这里声明一个`gen_fn`宏（代码详见下一节），在生成varint读的代码时，我们需要向`gen_fn`宏依次传入`fn_name`（方法名）、`type`（返回类型`u32`/`u64`）、`max_bytes`（varint最大字节）和`overflow_limit`（前文所说的`u32`/`u64`溢出值）；在生成varint写的代码时，我们需要向`gen_fn`宏依次传入`fn_name`（方法名）和`type`（写入数据类型`u32`/`u64`）。

### 完整proto-varint代码

本节将展示完整的`proto-varint`核心代码。

lib.rs：

```rust
use std::io::{Error, ErrorKind, Read, Write};

use zigzag::ZigZag;

mod zigzag;

const MAX_VARINT_32_BYTES: usize = 5;
const MAX_VARINT_64_BYTES: usize = 10;

macro_rules! gen_fn {
    ($fn_name:ident, $type:ty, $max_bytes:expr, $overflow_limit:expr) => {
        fn $fn_name(&mut self) -> Result<$type, Error> {
            let mut result = 0;
            let mut shift = 0;
            let mut buf = vec![0u8];
            for i in 0..$max_bytes {
                match self.read(&mut buf)? {
                    0 => return Err(Error::new(ErrorKind::UnexpectedEof, "EOF")),
                    1 => {
                        let v = buf[0];
                        if v < 0x80 {
                            if i == $max_bytes - 1 && v >= $overflow_limit {
                                return Err(Error::new(ErrorKind::InvalidData, "Overflow"));
                            }
                            return Ok(result | (v as $type) << shift);
                        }
                        result |= (v as $type & 0x7f) << shift;
                        shift += 7;
                    }
                    _ => {
                        return Err(Error::new(ErrorKind::InvalidData, "Invalid data"));
                    }
                }
            }

            Err(Error::new(ErrorKind::InvalidData, "Overflow"))
        }
    };
    ($fn_name:ident, $type:ty) => {
        fn $fn_name(&mut self, value: $type) -> Result<(), Error> {
            let mut value = value;
            while value >= 0x80 {
                self.write_all(&[value as u8 | 0x80])?;
                value >>= 7;
            }

            return self.write_all(&[value as u8]);
        }
    };
}

/// Read varint from source.
pub trait VarintRead: Read {
    fn read_signed_varint_32(&mut self) -> Result<i32, Error> {
        let v = self.read_unsigned_varint_32()?;
        Ok(v.zigzag())
    }

    gen_fn!(read_unsigned_varint_32, u32, MAX_VARINT_32_BYTES, 0x10);

    fn read_signed_varint_64(&mut self) -> Result<i64, Error> {
        let v = self.read_unsigned_varint_64()?;
        Ok(v.zigzag())
    }

    gen_fn!(read_unsigned_varint_64, u64, MAX_VARINT_64_BYTES, 0x02);
}

/// Write varint to a sink.
pub trait VarintWrite: Write {
    fn write_signed_varint_32(&mut self, value: i32) -> Result<(), Error> {
        self.write_unsigned_varint_32(value.zigzag())
    }

    gen_fn!(write_unsigned_varint_32, u32);

    fn write_signed_varint_64(&mut self, value: i64) -> Result<(), Error> {
        self.write_unsigned_varint_64(value.zigzag())
    }

    gen_fn!(write_unsigned_varint_64, u64);
}

// Implement VarintRead for any type that implements std::io::Read.
impl<T: Read> VarintRead for T {}
// Implement VarintWrite for any type that implements `std::io::Write`
impl<T: Write> VarintWrite for T {}
```

zigzag.rs：

```rust
pub trait ZigZag<T> {
    fn zigzag(&self) -> T;
}

impl ZigZag<u32> for i32 {
    fn zigzag(&self) -> u32 {
        ((self << 1) ^ (self >> 31)) as u32
    }
}

impl ZigZag<i32> for u32 {
    fn zigzag(&self) -> i32 {
        ((self >> 1) as i32) ^ -((self & 1) as i32)
    }
}

impl ZigZag<u64> for i64 {
    fn zigzag(&self) -> u64 {
        ((self << 1) ^ (self >> 63)) as u64
    }
}

impl ZigZag<i64> for u64 {
    fn zigzag(&self) -> i64 {
        ((self >> 1) as i64) ^ -((self & 1) as i64)
    }
}
```

### 单元测试Cases

Zig-Zag相关cases：

```rust
#[cfg(test)]
mod test {
    use super::ZigZag;

    #[test]
    fn test_zigzag_32_encode() {
        assert_eq!(0, 0i32.zigzag());
        assert_eq!(1, (-1i32).zigzag());
        assert_eq!(2, 1i32.zigzag());
        assert_eq!(3, (-2i32).zigzag());
        assert_eq!(0xfffffffe, 0x7fffffffi32.zigzag());
    }

    #[test]
    fn test_zigzag_32_decode() {
        assert_eq!(0, 0u32.zigzag());
        assert_eq!(-1, 1u32.zigzag());
        assert_eq!(1, 2u32.zigzag());
        assert_eq!(-2, 3u32.zigzag());
        assert_eq!(0x7fffffff, 0xfffffffeu32.zigzag());
        assert_eq!(-0x80000000, 0xffffffffu32.zigzag());
    }

    #[test]
    fn test_zigzag_64_encode() {
        assert_eq!(0, 0i64.zigzag());
        assert_eq!(1, (-1i64).zigzag());
        assert_eq!(2, 1i64.zigzag());
        assert_eq!(3, (-2i64).zigzag());
        assert_eq!(0xfffffffffffffffe, 0x7fffffffffffffffi64.zigzag());
    }

    #[test]
    fn test_zigzag_64_decode() {
        assert_eq!(0, 0u64.zigzag());
        assert_eq!(-1, 1u64.zigzag());
        assert_eq!(1, 2u64.zigzag());
        assert_eq!(-2, 3u64.zigzag());
        assert_eq!(0x7fffffffffffffff, 0xfffffffffffffffeu64.zigzag());
        assert_eq!(-0x8000000000000000, 0xffffffffffffffffu64.zigzag());
    }
}
```

Varint读写相关Cases，包含临界值、特殊值的Cases：

```rust
#[cfg(test)]
mod test {
    use std::io::{Cursor, ErrorKind};

    use crate::{VarintRead, VarintWrite};

    #[test]
    fn test_read_unsigned_varint_32() {
        let mut buf = Cursor::new(vec![0x01]);
        assert_eq!(Some(1), buf.read_unsigned_varint_32().ok());

        buf = Cursor::new(vec![0xB4, 0x01]);
        assert_eq!(Some(180), buf.read_unsigned_varint_32().ok());

        buf = Cursor::new(vec![0xB4, 0x81, 0x01]);
        assert_eq!(Some(16564), buf.read_unsigned_varint_32().ok());

        buf = Cursor::new(vec![0xB4, 0x81, 0x81, 0x01]);
        assert_eq!(Some(2113716), buf.read_unsigned_varint_32().ok());

        buf = Cursor::new(vec![0xB4, 0x81, 0x81, 0x81, 0x01]);
        assert_eq!(Some(270549172), buf.read_unsigned_varint_32().ok());

        buf = Cursor::new(vec![0xFF, 0xFF, 0xFF, 0xFF, 0x0F]);
        assert_eq!(Some(u32::MAX), buf.read_unsigned_varint_32().ok());
    }

    #[test]
    fn test_read_signed_varint_32() {
        let mut buf = Cursor::new(vec![0x01]);
        assert_eq!(Some(-1), buf.read_signed_varint_32().ok());

        buf = Cursor::new(vec![0x02]);
        assert_eq!(Some(1), buf.read_signed_varint_32().ok());

        buf = Cursor::new(vec![0x81, 0x01]);
        assert_eq!(Some(-65), buf.read_signed_varint_32().ok());

        buf = Cursor::new(vec![0x82, 0x01]);
        assert_eq!(Some(65), buf.read_signed_varint_32().ok());

        buf = Cursor::new(vec![0x81, 0x81, 0x01]);
        assert_eq!(Some(-8257), buf.read_signed_varint_32().ok());

        buf = Cursor::new(vec![0x82, 0x81, 0x01]);
        assert_eq!(Some(8257), buf.read_signed_varint_32().ok());

        buf = Cursor::new(vec![0x81, 0x81, 0x81, 0x01]);
        assert_eq!(Some(-1056833), buf.read_signed_varint_32().ok());

        buf = Cursor::new(vec![0x82, 0x81, 0x81, 0x01]);
        assert_eq!(Some(1056833), buf.read_signed_varint_32().ok());

        buf = Cursor::new(vec![0x81, 0x81, 0x81, 0x81, 0x01]);
        assert_eq!(Some(-135274561), buf.read_signed_varint_32().ok());

        buf = Cursor::new(vec![0x82, 0x81, 0x81, 0x81, 0x01]);
        assert_eq!(Some(135274561), buf.read_signed_varint_32().ok());

        buf = Cursor::new(vec![0xFF, 0xFF, 0xFF, 0xFF, 0x0F]);
        assert_eq!(Some(i32::MIN), buf.read_signed_varint_32().ok());

        buf = Cursor::new(vec![0xFE, 0xFF, 0xFF, 0xFF, 0x0F]);
        assert_eq!(Some(i32::MAX), buf.read_signed_varint_32().ok());
    }

    #[test]
    fn test_read_write_unsigned_varint_32() {
        let mut buf = Cursor::new(vec![0u8; 0]);
        assert!(buf.write_unsigned_varint_32(1).is_ok());
        buf.set_position(0);
        assert_eq!(Some(1), buf.read_unsigned_varint_32().ok());

        buf.set_position(0);
        assert!(buf.write_unsigned_varint_32(180).is_ok());
        buf.set_position(0);
        assert_eq!(Some(180), buf.read_unsigned_varint_32().ok());

        buf.set_position(0);
        assert!(buf.write_unsigned_varint_32(16564).is_ok());
        buf.set_position(0);
        assert_eq!(Some(16564), buf.read_unsigned_varint_32().ok());

        buf.set_position(0);
        assert!(buf.write_unsigned_varint_32(2113716).is_ok());
        buf.set_position(0);
        assert_eq!(Some(2113716), buf.read_unsigned_varint_32().ok());
    }

    #[test]
    fn test_read_write_signed_varint_32() {
        let mut buf = Cursor::new(vec![0u8; 0]);
        assert!(buf.write_signed_varint_32(-1).is_ok());
        buf.set_position(0);
        assert_eq!(Some(-1), buf.read_signed_varint_32().ok());

        buf.set_position(0);
        assert!(buf.write_signed_varint_32(1).is_ok());
        buf.set_position(0);
        assert_eq!(Some(1), buf.read_signed_varint_32().ok());

        buf.set_position(0);
        assert!(buf.write_signed_varint_32(-65).is_ok());
        buf.set_position(0);
        assert_eq!(Some(-65), buf.read_signed_varint_32().ok());

        buf.set_position(0);
        assert!(buf.write_signed_varint_32(65).is_ok());
        buf.set_position(0);
        assert_eq!(Some(65), buf.read_signed_varint_32().ok());

        buf.set_position(0);
        assert!(buf.write_signed_varint_32(-8257).is_ok());
        buf.set_position(0);
        assert_eq!(Some(-8257), buf.read_signed_varint_32().ok());

        buf.set_position(0);
        assert!(buf.write_signed_varint_32(8257).is_ok());
        buf.set_position(0);
        assert_eq!(Some(8257), buf.read_signed_varint_32().ok());

        buf.set_position(0);
        assert!(buf.write_signed_varint_32(-1056833).is_ok());
        buf.set_position(0);
        assert_eq!(Some(-1056833), buf.read_signed_varint_32().ok());

        buf.set_position(0);
        assert!(buf.write_signed_varint_32(1056833).is_ok());
        buf.set_position(0);
        assert_eq!(Some(1056833), buf.read_signed_varint_32().ok());

        buf.set_position(0);
        assert!(buf.write_signed_varint_32(-135274561).is_ok());
        buf.set_position(0);
        assert_eq!(Some(-135274561), buf.read_signed_varint_32().ok());

        buf.set_position(0);
        assert!(buf.write_signed_varint_32(135274561).is_ok());
        buf.set_position(0);
        assert_eq!(Some(135274561), buf.read_signed_varint_32().ok());
    }

    #[test]
    fn test_read_unsigned_varint_64() {
        let mut buf = Cursor::new(vec![0x01]);
        assert_eq!(Some(1), buf.read_unsigned_varint_64().ok());

        buf = Cursor::new(vec![0xB4, 0x01]);
        assert_eq!(Some(180), buf.read_unsigned_varint_64().ok());

        buf = Cursor::new(vec![0xB4, 0x81, 0x01]);
        assert_eq!(Some(16564), buf.read_unsigned_varint_64().ok());

        buf = Cursor::new(vec![0xB4, 0x81, 0x81, 0x01]);
        assert_eq!(Some(2113716), buf.read_unsigned_varint_64().ok());

        buf = Cursor::new(vec![0xB4, 0x81, 0x81, 0x81, 0x01]);
        assert_eq!(Some(270549172), buf.read_unsigned_varint_64().ok());

        buf = Cursor::new(vec![0xB4, 0x81, 0x81, 0x81, 0x81, 0x01]);
        assert_eq!(Some(34630287540), buf.read_unsigned_varint_64().ok());

        buf = Cursor::new(vec![0xB4, 0x81, 0x81, 0x81, 0x81, 0x81, 0x01]);
        assert_eq!(Some(4432676798644), buf.read_unsigned_varint_64().ok());

        buf = Cursor::new(vec![0xB4, 0x81, 0x81, 0x81, 0x81, 0x81, 0x81, 0x01]);
        assert_eq!(Some(567382630219956), buf.read_unsigned_varint_64().ok());

        buf = Cursor::new(vec![0xB4, 0x81, 0x81, 0x81, 0x81, 0x81, 0x81, 0x81, 0x01]);
        assert_eq!(Some(72624976668147892), buf.read_unsigned_varint_64().ok());

        buf = Cursor::new(vec![
            0xB4, 0x81, 0x81, 0x81, 0x81, 0x81, 0x81, 0x81, 0x81, 0x01,
        ]);
        assert_eq!(
            Some(9295997013522923700),
            buf.read_unsigned_varint_64().ok()
        );

        buf = Cursor::new(vec![
            0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0x01,
        ]);
        assert_eq!(Some(u64::MAX), buf.read_unsigned_varint_64().ok());
    }

    #[test]
    fn test_read_signed_varint_64() {
        let mut buf = Cursor::new(vec![0x01]);
        assert_eq!(Some(-1), buf.read_signed_varint_64().ok());

        buf = Cursor::new(vec![0x02]);
        assert_eq!(Some(1), buf.read_signed_varint_64().ok());

        buf = Cursor::new(vec![0x81, 0x01]);
        assert_eq!(Some(-65), buf.read_signed_varint_64().ok());

        buf = Cursor::new(vec![0x82, 0x01]);
        assert_eq!(Some(65), buf.read_signed_varint_64().ok());

        buf = Cursor::new(vec![0x81, 0x81, 0x01]);
        assert_eq!(Some(-8257), buf.read_signed_varint_64().ok());

        buf = Cursor::new(vec![0x82, 0x81, 0x01]);
        assert_eq!(Some(8257), buf.read_signed_varint_64().ok());

        buf = Cursor::new(vec![0x81, 0x81, 0x81, 0x01]);
        assert_eq!(Some(-1056833), buf.read_signed_varint_64().ok());

        buf = Cursor::new(vec![0x82, 0x81, 0x81, 0x01]);
        assert_eq!(Some(1056833), buf.read_signed_varint_64().ok());

        buf = Cursor::new(vec![0x81, 0x81, 0x81, 0x81, 0x01]);
        assert_eq!(Some(-135274561), buf.read_signed_varint_64().ok());

        buf = Cursor::new(vec![0x82, 0x81, 0x81, 0x81, 0x01]);
        assert_eq!(Some(135274561), buf.read_signed_varint_64().ok());

        buf = Cursor::new(vec![0x81, 0x81, 0x81, 0x81, 0x81, 0x01]);
        assert_eq!(Some(-17315143745), buf.read_signed_varint_64().ok());

        buf = Cursor::new(vec![0x82, 0x81, 0x81, 0x81, 0x81, 0x01]);
        assert_eq!(Some(17315143745), buf.read_signed_varint_64().ok());

        buf = Cursor::new(vec![0x81, 0x81, 0x81, 0x81, 0x81, 0x81, 0x01]);
        assert_eq!(Some(-2216338399297), buf.read_signed_varint_64().ok());

        buf = Cursor::new(vec![0x82, 0x81, 0x81, 0x81, 0x81, 0x81, 0x01]);
        assert_eq!(Some(2216338399297), buf.read_signed_varint_64().ok());

        buf = Cursor::new(vec![0x81, 0x81, 0x81, 0x81, 0x81, 0x81, 0x81, 0x01]);
        assert_eq!(Some(-283691315109953), buf.read_signed_varint_64().ok());

        buf = Cursor::new(vec![0x82, 0x81, 0x81, 0x81, 0x81, 0x81, 0x81, 0x01]);
        assert_eq!(Some(283691315109953), buf.read_signed_varint_64().ok());

        buf = Cursor::new(vec![0x81, 0x81, 0x81, 0x81, 0x81, 0x81, 0x81, 0x81, 0x01]);
        assert_eq!(Some(-36312488334073921), buf.read_signed_varint_64().ok());

        buf = Cursor::new(vec![0x82, 0x81, 0x81, 0x81, 0x81, 0x81, 0x81, 0x81, 0x01]);
        assert_eq!(Some(36312488334073921), buf.read_signed_varint_64().ok());

        buf = Cursor::new(vec![
            0x81, 0x81, 0x81, 0x81, 0x81, 0x81, 0x81, 0x81, 0x81, 0x01,
        ]);
        assert_eq!(Some(-4647998506761461825), buf.read_signed_varint_64().ok());

        buf = Cursor::new(vec![
            0x82, 0x81, 0x81, 0x81, 0x81, 0x81, 0x81, 0x81, 0x81, 0x01,
        ]);
        assert_eq!(Some(4647998506761461825), buf.read_signed_varint_64().ok());

        buf = Cursor::new(vec![
            0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0x01,
        ]);
        assert_eq!(Some(i64::MIN), buf.read_signed_varint_64().ok());

        buf = Cursor::new(vec![
            0xFE, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0x01,
        ]);
        assert_eq!(Some(i64::MAX), buf.read_signed_varint_64().ok());
    }

    #[test]
    fn test_read_write_unsigned_varint_64() {
        let mut buf = Cursor::new(vec![0u8; 0]);
        assert!(buf.write_unsigned_varint_64(1).is_ok());
        buf.set_position(0);
        assert_eq!(Some(1), buf.read_unsigned_varint_64().ok());

        buf.set_position(0);
        assert!(buf.write_unsigned_varint_64(180).is_ok());
        buf.set_position(0);
        assert_eq!(Some(180), buf.read_unsigned_varint_64().ok());

        buf.set_position(0);
        assert!(buf.write_unsigned_varint_64(16564).is_ok());
        buf.set_position(0);
        assert_eq!(Some(16564), buf.read_unsigned_varint_64().ok());

        buf.set_position(0);
        assert!(buf.write_unsigned_varint_64(2113716).is_ok());
        buf.set_position(0);
        assert_eq!(Some(2113716), buf.read_unsigned_varint_64().ok());

        buf.set_position(0);
        assert!(buf.write_unsigned_varint_64(270549172).is_ok());
        buf.set_position(0);
        assert_eq!(Some(270549172), buf.read_unsigned_varint_64().ok());

        buf.set_position(0);
        assert!(buf.write_unsigned_varint_64(34630287540).is_ok());
        buf.set_position(0);
        assert_eq!(Some(34630287540), buf.read_unsigned_varint_64().ok());

        buf.set_position(0);
        assert!(buf.write_unsigned_varint_64(4432676798644).is_ok());
        buf.set_position(0);
        assert_eq!(Some(4432676798644), buf.read_unsigned_varint_64().ok());

        buf.set_position(0);
        assert!(buf.write_unsigned_varint_64(567382630219956).is_ok());
        buf.set_position(0);
        assert_eq!(Some(567382630219956), buf.read_unsigned_varint_64().ok());

        buf.set_position(0);
        assert!(buf.write_unsigned_varint_64(72624976668147892).is_ok());
        buf.set_position(0);
        assert_eq!(Some(72624976668147892), buf.read_unsigned_varint_64().ok());

        buf.set_position(0);
        assert!(buf.write_unsigned_varint_64(9295997013522923700).is_ok());
        buf.set_position(0);
        assert_eq!(
            Some(9295997013522923700),
            buf.read_unsigned_varint_64().ok()
        );
    }

    #[test]
    fn test_read_write_signed_varint_64() {
        let mut buf = Cursor::new(vec![0u8; 0]);
        assert!(buf.write_signed_varint_64(-1).is_ok());
        buf.set_position(0);
        assert_eq!(Some(-1), buf.read_signed_varint_64().ok());

        buf.set_position(0);
        assert!(buf.write_signed_varint_64(1).is_ok());
        buf.set_position(0);
        assert_eq!(Some(1), buf.read_signed_varint_64().ok());

        buf.set_position(0);
        assert!(buf.write_signed_varint_64(-65).is_ok());
        buf.set_position(0);
        assert_eq!(Some(-65), buf.read_signed_varint_64().ok());

        buf.set_position(0);
        assert!(buf.write_signed_varint_64(65).is_ok());
        buf.set_position(0);
        assert_eq!(Some(65), buf.read_signed_varint_64().ok());

        buf.set_position(0);
        assert!(buf.write_signed_varint_64(-8257).is_ok());
        buf.set_position(0);
        assert_eq!(Some(-8257), buf.read_signed_varint_64().ok());

        buf.set_position(0);
        assert!(buf.write_signed_varint_64(8257).is_ok());
        buf.set_position(0);
        assert_eq!(Some(8257), buf.read_signed_varint_64().ok());

        buf.set_position(0);
        assert!(buf.write_signed_varint_64(-1056833).is_ok());
        buf.set_position(0);
        assert_eq!(Some(-1056833), buf.read_signed_varint_64().ok());

        buf.set_position(0);
        assert!(buf.write_signed_varint_64(1056833).is_ok());
        buf.set_position(0);
        assert_eq!(Some(1056833), buf.read_signed_varint_64().ok());

        buf.set_position(0);
        assert!(buf.write_signed_varint_64(-135274561).is_ok());
        buf.set_position(0);
        assert_eq!(Some(-135274561), buf.read_signed_varint_64().ok());

        buf.set_position(0);
        assert!(buf.write_signed_varint_64(135274561).is_ok());
        buf.set_position(0);
        assert_eq!(Some(135274561), buf.read_signed_varint_64().ok());

        buf.set_position(0);
        assert!(buf.write_signed_varint_64(-17315143745).is_ok());
        buf.set_position(0);
        assert_eq!(Some(-17315143745), buf.read_signed_varint_64().ok());

        buf.set_position(0);
        assert!(buf.write_signed_varint_64(17315143745).is_ok());
        buf.set_position(0);
        assert_eq!(Some(17315143745), buf.read_signed_varint_64().ok());

        buf.set_position(0);
        assert!(buf.write_signed_varint_64(-2216338399297).is_ok());
        buf.set_position(0);
        assert_eq!(Some(-2216338399297), buf.read_signed_varint_64().ok());

        buf.set_position(0);
        assert!(buf.write_signed_varint_64(2216338399297).is_ok());
        buf.set_position(0);
        assert_eq!(Some(2216338399297), buf.read_signed_varint_64().ok());

        buf.set_position(0);
        assert!(buf.write_signed_varint_64(-283691315109953).is_ok());
        buf.set_position(0);
        assert_eq!(Some(-283691315109953), buf.read_signed_varint_64().ok());

        buf.set_position(0);
        assert!(buf.write_signed_varint_64(283691315109953).is_ok());
        buf.set_position(0);
        assert_eq!(Some(283691315109953), buf.read_signed_varint_64().ok());

        buf.set_position(0);
        assert!(buf.write_signed_varint_64(-36312488334073921).is_ok());
        buf.set_position(0);
        assert_eq!(Some(-36312488334073921), buf.read_signed_varint_64().ok());

        buf.set_position(0);
        assert!(buf.write_signed_varint_64(36312488334073921).is_ok());
        buf.set_position(0);
        assert_eq!(Some(36312488334073921), buf.read_signed_varint_64().ok());

        buf.set_position(0);
        assert!(buf.write_signed_varint_64(-4647998506761461825).is_ok());
        buf.set_position(0);
        assert_eq!(Some(-4647998506761461825), buf.read_signed_varint_64().ok());

        buf.set_position(0);
        assert!(buf.write_signed_varint_64(4647998506761461825).is_ok());
        buf.set_position(0);
        assert_eq!(Some(4647998506761461825), buf.read_signed_varint_64().ok());
    }

    #[test]
    fn test_read_varint_overflow() {
        let mut buf = Cursor::new(vec![0x96, 0x81, 0x81, 0x81, 0x81, 0x01]);
        assert!(buf
            .read_unsigned_varint_32()
            .is_err_and(|e| e.kind() == ErrorKind::InvalidData));

        buf = Cursor::new(vec![0x96, 0x81, 0x81, 0x81, 0x81, 0x01]);
        assert!(buf
            .read_signed_varint_32()
            .is_err_and(|e| e.kind() == ErrorKind::InvalidData));

        buf = Cursor::new(vec![0x96, 0x81, 0x81, 0x81, 0x10]);
        assert!(buf
            .read_unsigned_varint_32()
            .is_err_and(|e| e.kind() == ErrorKind::InvalidData));

        buf = Cursor::new(vec![0x96, 0x81, 0x81, 0x81, 0x10]);
        assert!(buf
            .read_signed_varint_32()
            .is_err_and(|e| e.kind() == ErrorKind::InvalidData));

        buf = Cursor::new(vec![
            0x96, 0x81, 0x81, 0x81, 0x81, 0x81, 0x81, 0x81, 0x81, 0x81, 0x01,
        ]);
        assert!(buf
            .read_unsigned_varint_64()
            .is_err_and(|e| e.kind() == ErrorKind::InvalidData));

        buf = Cursor::new(vec![
            0x96, 0x81, 0x81, 0x81, 0x81, 0x81, 0x81, 0x81, 0x81, 0x81, 0x01,
        ]);
        assert!(buf
            .read_signed_varint_64()
            .is_err_and(|e| e.kind() == ErrorKind::InvalidData));

        buf = Cursor::new(vec![
            0x96, 0x81, 0x81, 0x81, 0x81, 0x81, 0x81, 0x81, 0x81, 0x02,
        ]);
        assert!(buf
            .read_unsigned_varint_64()
            .is_err_and(|e| e.kind() == ErrorKind::InvalidData));

        buf = Cursor::new(vec![
            0x96, 0x81, 0x81, 0x81, 0x81, 0x81, 0x81, 0x81, 0x81, 0x02,
        ]);
        assert!(buf
            .read_signed_varint_64()
            .is_err_and(|e| e.kind() == ErrorKind::InvalidData));
    }

    #[test]
    fn test_read_write_many_unsigned_varint_32() {
        let mut vector = Cursor::new(vec![0u8; 0]);
        for i in 0..1 << 20 {
            vector.write_unsigned_varint_32(i).unwrap();
        }

        vector.set_position(0);
        for i in 0..1 << 20 {
            assert_eq!(Some(i), vector.read_unsigned_varint_32().ok());
        }
    }

    #[test]
    fn test_read_write_many_signed_varint_32() {
        let mut vector = Cursor::new(vec![0u8; 0]);
        for i in -(1 << 19)..1 << 19 {
            vector.write_signed_varint_32(i).unwrap();
        }

        vector.set_position(0);
        for i in -(1 << 19)..1 << 19 {
            assert_eq!(Some(i), vector.read_signed_varint_32().ok());
        }
    }

    #[test]
    fn test_read_write_many_unsigned_varint_64() {
        let mut vector = Cursor::new(vec![0u8; 0]);
        for i in 0..1 << 20 {
            vector.write_unsigned_varint_64(i).unwrap();
        }

        vector.set_position(0);
        for i in 0..1 << 20 {
            assert_eq!(Some(i), vector.read_unsigned_varint_64().ok());
        }
    }

    #[test]
    fn test_read_write_many_signed_varint_64() {
        let mut vector = Cursor::new(vec![0u8; 0]);
        for i in -(1 << 19)..1 << 19 {
            vector.write_signed_varint_64(i).unwrap();
        }

        vector.set_position(0);
        for i in -(1 << 19)..1 << 19 {
            assert_eq!(Some(i), vector.read_signed_varint_64().ok());
        }
    }
}
```
