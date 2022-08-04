---
title: "「力扣刷题」8.字符串转换整数 (atoi)"
date: 2022-08-03T23:34:09+08:00
draft: false
---

[这道题](https://leetcode.cn/problems/string-to-integer-atoi/)是一道考察字符串的中等难度题目，首先来看一下这道题的题干：

![8](/images/Leetcode/8/leetcode-8.png)

这道题如果用库函数做那就是一行代码的事，但是题目显然是不推荐使用库函数的。但尽管不用库函数，这道题乍一看也觉得很简单，最初读完题干的感觉就是虽然题目有很多要求，但是它都给你一一总结列举了出来，所以只需要多写几个if else满足这些要求就好啦！看我分分钟写完它！

但是写着写着就会发现我的代码越来越臃肿，感觉有很多if-else可以被优化但是又不敢轻易修改。这还不是最头疼的，第一次点击提交：解答错误！第二次点击提交：解答错误，第三次……每一次提交，都会看到还有一些情况没有考虑到，但是考虑了这些情况后之前的测试用例又不通过了，于是折腾来折腾去就有了如下的惨状：

![fails](/images/Leetcode/8/fails.png)

其实冷静一下再仔细读读题干，然后我们假设有一个游标`c`指向输入字符串`s`的一个字符，`c`只存在以下情况：

- 是空格
- 是+或者-
- 是数字
- 是其它字符

当`c`从`s`的第一个字符向最后一个字符进行滑动时候，我们假设有一个状态`state`，它也只有这几种情况：

- 还没开始读取数字
- 确定数字的正负
- 读取数字的绝对值
- 结束

而且不同`state`之间的转换是取决于上面说的`c`的情况的，所以这么一来，这道题就可以用**状态转移**的思想来进行解决，根据我们总结的`c`和`state`以及题目要求，我们就可以绘制出下面这样的一个状态转移图：

![automaton](/images/Leetcode/8/automaton.jpg)

实际上这种思想在计算机学术界有一个这样的名字——[**确定有限状态自动机**](https://baike.baidu.com/item/确定有限状态自动机/22722750?fr=aladdin)，对于一个给定的属于该自动机的状态和一个属于该自动机字母表求和的字符，它都能根据事先给定的转移函数转移到下一个状态（这个状态可以是先前那个状态）

根据状态转移图，我们可以定义出状态类型，和状态的转移表：

```go
type State uint8

// 四种状态
const (
	StateStart = iota
	StateSigned
	StateNumber
	StateEnd
)

// stateTable 状态转移表
var stateTable = [4][4]State{
	StateStart:  {StateStart, StateSigned, StateNumber, StateEnd},
	StateSigned: {StateEnd, StateEnd, StateNumber, StateEnd},
	StateNumber: {StateEnd, StateEnd, StateNumber, StateEnd},
	StateEnd:    {StateEnd, StateEnd, StateEnd, StateEnd},
}
```

我们假设状态转移表的第一列是当输入`c`为空格时对应的输出状态；第二列是当输入`c`为+/-时对应的输出状态；第三列是当输入`c`为数字时对应的输出状态；第四列是当输入`c`为其它字符时对应的输出状态。因此，我们给`State`类型定义一个`Next`方法，根据输入`c`得出输出状态：

```go
func (s State) Next(c byte) State {
	if c == ' ' {
		return stateTable[s][0]
	}
	if c == '+' || c == '-' {
		return stateTable[s][1]
	}
	if c >= '0' && c <= '9' {
		return stateTable[s][2]
	}
	return stateTable[s][3]
}
```

接下来我们就要定义一个自动机`Automaton`了，它能够根据输入`c`，然后改变自己内部的状态`state`，同时还需要记录当前读取出数字的结果。因此我们定义了一个`Automaton`接口类型，它有两个方法，定义如下：

```go
// Automaton 自动机接口
type Automaton interface {
	Process(c byte) bool
	Res() int
}
```

这里`Process`方法有一个`bool`类型的返回值，用于告诉调用者还需不需要继续进行读取，因为题目要求里写了直到到达下一个非数字字符或到达输入的结尾，字符串的其余部分将被忽略，也就是说`StateEnd`无论给它什么输入`c`都会输出`StateEnd`，所以遇到了`StateEnd`就可以不用继续`Process`了。这样，我们主程序就可以这样调用：

```go
func myAtoi(s string) int {
	atmt := NewAutomaton()
	for i := range s {
		if !atmt.Process(s[i]) {
			break
		}
	}

	return atmt.Res()
}
```

我们目前还没有对`Automaton`进行实现，因此我们定义`automatonImpl`实现该接口。`automatonImpl`字段定义如下：

```go
type automatonImpl struct {
	state  State // 状态
	abs    int   // 读取的数字绝对值
	signed int   // 符号
	res    int32 // 读取的数字结果
}
```

实现`Process`方法：

```go
func (a *automatonImpl) Process(c byte) bool {
	a.state = a.state.Next(c) // 根据输入c更新自动机状态
	switch a.state {
	case StateSigned:
        // 判定符号
		if c == '-' {
			a.signed = -1
		} else {
			a.signed = 1
		}
	case StateNumber:
        // 更新读取数字的绝对值
		a.abs = 10*a.abs + int(c-'0')
        
        // 由于题目要求返回的结果是32位的，所以这里要进行判断看看是否溢出，按照题目要求，如果溢出则要进行截断
		tmp := a.abs * a.signed
		if tmp > math.MaxInt32 {
			a.abs = math.MaxInt32
		} else if tmp < math.MinInt32 {
			a.abs = -math.MinInt32
		}
		a.res = int32(a.abs * a.signed)
	case StateEnd:
        // 如果是StateEnd就通知调用者可以不用继续Process了
		return false
	default:
	}

	return true
}

func (a *automatonImpl) Res() int {
	return int(a.res)
}
```

最终，我们再写一个`NewAutomaton`函数用于初始化自动机，并返回：

```go
func NewAutomaton() Automaton {
	return &automatonImpl{
		state:  StateStart, // 初始状态
		abs:    0,
		signed: 1,
	}
}
```

这样一来，我们的代码就没那么臃肿了，而且有着良好的扩展性、抽象性。最后点击提交，完美！

![allpass](/images/Leetcode/8/allpass.png)
