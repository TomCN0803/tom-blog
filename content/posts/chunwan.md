---
title: "一文揭秘刘谦春晚魔术背后到底发生了什么"
date: 2024-02-10T12:00:00+08:00
draft: false
---

刘谦老师在今年央视龙年春晚的魔术表演绝对是是最精彩的节目之一，尤其是第二个魔术，创新性地让现场、电视机前的观众都参与到了魔术表演中，直观感受到了魔术表演的神奇。由于全体观众都有参与，所以这个魔术不需要任何花哨的魔术手法，其背后就是道综合的数学问题。

## 约瑟夫环问题

在进行该魔术原理介绍之前，我们先了解一个与其相关的重要数学问题——约瑟夫环问题。

### 传说故事

约瑟夫环问题的起源可以追溯到公元 1 世纪。据传，在当时有一个名叫弗拉维奥·约瑟夫的犹太历史学家，有一次他和他的 40 个战友被罗马军队包围，在面临绝境时，他们决定不做俘虏，于是决定通过自杀的方式结束生命。为了执行这个决定，他们围成一个圈，然后按照一定的规则来选择自杀的人，直到只剩下最后一个人自己完成自杀。

大致规则如下：战友们站立围成一个圆圈，计数从圆圈中的指定点开始，并沿指定方向围绕圆圈进行。在跳过指定数量的人之后，处刑下一个人。对剩下的人重复该过程，从下一个人开始，朝同一方向跳过相同数量的人，直到只剩下一最后一个存活的人。

而约瑟夫，作为一个不愿意自杀的人，快速地计算出了一个位置，使得他成为了最后一个存活的人，从而有机会逃脱。

### 数学模型

如果我们将这个故事内容转换成数学模型，就得到了约瑟夫环问题：假设有`n`个人围成一圈，从某个人开始按照顺时针方向逐一进行编号，编号从`0`开始（因此有编号为`0`到`n-1`的人）。下面从编号为`0`的人开始报数，报数从`1`开始，每数到`k`，就将这个人从圈中排除，然后从下一个人重新开始从`1`报数，直到最后只剩下一个人。问题即，给定起始人数`n`、起点（也就是谁编号是`0`）、和`k`，给出最后剩下的那个人的编号。

举个具体例子，如果有`n=5`个人参与约瑟夫环，则有编号`0`到`4`，选择的`k=2`，那么游戏的过程是这样的：

1. 从编号为`0`的人开始报数，数到`2`的人是编号为`1`的人，所以编号为`1`的人离开游戏；
2. 从编号为`2`的人开始报数，数到`2`的人是编号为`3`的人，所以编号为`3`的人离开游戏；
3. 从编号为`4`的人开始报数，数到`2`的人是编号为`0`的人，所以编号为`0`的人离开游戏；
4. 从编号为`2`的人开始报数，数到`2`的人是编号为`4`的人，所以编号为`4`的人离开游戏；
5. 最终剩下编号为`2`的人，得出答案`2`。

### 解决思路

我们现令`F(n, k)`来表示约瑟夫环问题，那么上一小节的例子就是`F(5, 2)`。以该`F(5, 2)`为例，绘制下面这张图：我们使用`A`到`E`分别来给每个人进行编号，而`0`至`4`的索引位置固定。由上小节知道，`F(5, 2) = 2`，也就是绿色标记的`C`是最后存活下来的人。我们再用黄色标记本轮报数将淘汰的人。我们可以有以下结论：**每一轮的游戏，都相当于上一轮游戏结束后，从被淘汰的人的下一个人所开始的人数减`1`的约瑟夫环问题**。为了方便理解，我们将每一轮游戏第一个人都对齐到索引`0`：

![约瑟夫环F(5, 2)问题](https://files.mdnice.com/user/17908/0690bc26-bca4-4582-b9ad-621e12136e1f.png)

因此我们可以用递归的思想来解决约瑟夫环问题。由于`n=1`时，也就是`F(1, k)`一定是`0`，因此其可以作为 base case。那么对于一般情况，当前轮游戏结果如何与上一轮游戏结果产生联系呢？例如`F(3, 2)`与`F(4, 2)`，首先要将`F(4, 2)`中被淘汰的`A`补回`F(3, 2)`中；然后将`F(3, 2)`右移两个索引，也就是`k`个索引，使`C`对齐；`C`的索引此时是`4`，由于约瑟夫环中的选手都是以呈环形站立，且`F(4, 2)`最大索引是`3`，所以`C`会回到索引`0`，也就是`4 % 4`：

那么现在就给出约瑟夫环问题的公式：

$$
F(n, k) = \left\{
\begin{align}
 &0 &, n &= 1 \notag \\
 &[F(n-1) + k] \mod n &, n &\gt1 \notag
\end{align}\right.
$$

用代码表示就是这样（Rust 代码编写的`josephus`函数，`num`为总人数）：

```rust
// Rust代码编写的约瑟夫环问题函数，num为总人数，每轮游戏报数时数到target的人被淘汰
pub fn josephus(num: i32, target: i32) -> i32 {
    if num == 1 {
        0
    } else {
        (josephus(num - 1, target) + target) % num
    }
}
```

由于每一轮游戏都只与上一轮的游戏结果有关，因此`josephus`函数还可以再优化一下，使用 Rust 迭代器的`fold`一行搞定：

```rust
pub fn josephus(num: i32, target: i32) -> i32 {
    (2..=num).fold(0, |prev, n| (prev + target) % n)
}
```

如果你关于约瑟夫环问题还有更多思路，可以试试 LeetCode 上面的[破冰游戏](https://leetcode.cn/problems/yuan-quan-zhong-zui-hou-sheng-xia-de-shu-zi-lcof/description/ "破冰游戏")问题。

## 复盘刘谦的魔术

本节将对刘谦老师在春晚表演的魔术逐步进行解释。我们设开始的四张牌为`ABCD`。

开始前，将四张牌对折撕开后上下堆叠排列，形成周期为`4`的排列顺序，即`ABCDABCD`：

![撕牌](https://files.mdnice.com/user/17908/d1efdb5f-d4e6-4532-9181-2002a37fac9d.png)

**步骤一**：根据名字字数为次数，每次从牌顶移动一张卡牌到牌低。这里由于名字数量不确定，所以其实可以看作是移动任意次（图中以三个字名字为例），因为不论移动多少次，**牌堆中第`4`张与第`8`张（最后一张）一定是一样的**。原因也很简单，我们可以把`ABCDABCD`看作以`ABCD`为最小周期的函数 $f(x) = f(x + 4)$，根据周期函数性质 $f(x + N) = f(x + 4 + N)$，就可以得到上述结论：

![步骤一](https://files.mdnice.com/user/17908/126ac33d-aa10-47d5-885a-9d5e7b38762a.png)

**步骤二**：将牌堆中前三张牌插入牌堆中间，这时候牌堆的第`1`张和最后一张一定是一样的。这时候开始其它牌是什么对我们来说就已经不重要了：

![步骤二](https://files.mdnice.com/user/17908/6ac30e1d-f790-498a-9d7f-bfaa1873f41a.png)

**步骤三**：拿走第`1`张收起来，也就是和最后一张一样的那张牌，此时牌堆中剩下`7`张牌：

![步骤三](https://files.mdnice.com/user/17908/91e444ef-fae3-40bd-bb71-8c41cda13f33.png)

**步骤四**：根据南方人、北方人或不确定分别拿`1`、`2`、`3`张牌插入牌堆中间。由于这一步对最后一张牌没有影响，所以对结果也不会产生影响，属于增加表演效果的步骤。

**步骤五**：根据男生或女生的区别，分别从牌堆中拿走`1`、`2`张牌，然后下咒“见证奇迹的时刻”——进行七次从牌顶向牌底挪动卡牌的动作。这里对于男生手中的牌相当于把最后一张牌向上移动了 $7 \mod 6 = 1$ 个位置；对于女生手中的牌相当于把最后一张牌向上移动了 $7 \mod 5 = 2$ 个位置。

**步骤六**：最后表演的高潮，就是“幸福留下来，烦恼丢出去”，直到手里留下一张牌。这一步实际上就是`k=2`的约瑟夫环问题，对于男生就是`F(6, 2)`，对于女生就是`F(5, 2)`，上一步中最后一张牌的位置正好对应了这一步中约瑟夫环的答案，因此这一步结束后手里所剩下的牌一定是最后一张牌，自然与步骤三中收起来的那张牌是一样的。

![步骤五、六](https://files.mdnice.com/user/17908/778854e7-99b5-4814-a02d-9b2c48ebe002.png)

## 总结

本文首先介绍了约瑟夫环问题，给出了约瑟夫环的递归解决思路，并根据该思路写出了 Rust 代码的实现与优化；之后对刘谦老师春晚魔术表演进行了逐步的复盘，揭秘了魔术背后的数学原理。刘谦老师今年魔术的大获成功，离不开数学的奇妙，同样也更离不开刘谦老师对背后数学原理的包装设计，与深厚的魔术表演功底。当然还有小尼同学的“小穿帮”，更是给节目添加了欢乐。
