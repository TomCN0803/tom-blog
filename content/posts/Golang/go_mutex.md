---
title: "「Go系列」互斥锁Mutex深入学习"
date: 2022-08-20T11:06:38+08:00
draft: true

---

`Mutext`数据结构如下，因为里面的变量都是整型，所以零值就是一个有效的互斥锁，处于Unlock状态：

```go
type Mutex struct {
   state int32
   sema  uint32
}
```

- state：互斥锁的状态，在Mutex加锁和解锁时候都是通过atomic包中的原子操作对state进行操作；
- sema：当前Mutex的信号量，具体实现锁、锁的等待队列。

## Mutex的模式

### 正常模式

- 加锁时如果没有获得锁，先尝试自旋几次（`active_spin=4`），如果自旋几次仍然获得不了锁，那么就通过信号量让该goroutine排队等待，按照FIFO的顺序；
- 一个goroutine释放锁的时候会唤醒等待队列中队首的goroutine，如果被唤醒的goroutine没有竞争过新来的在自旋的goroutine，那么就会再次被重新放到等待队列队首。

优点：

- **可以提高吞吐量**：新来的goroutine占有CPU时间片，如果能在自旋阶段拿到锁，就不需要被睡眠和后续唤醒，睡眠和唤醒goroutine是很耽误时间的。

缺点：

- **造成队尾延迟**：一个goroutine释放锁的时候会唤醒等待队列中队首的goroutine，但刚唤醒的goroutine难免竞争不过新来的拥有CPU时间片的goroutine，这样会造成等待队列中goroutine饥饿。

#### 自旋条件

通过[`sync_runtime_canSpin`方法](https://github.com/golang/go/blob/846dce9d05f19a1f53465e62a304dea21b99f910/src/runtime/proc.go#L5580)来判断是否满足自旋条件：

```go
func sync_runtime_canSpin(i int) bool {
	if i >= active_spin || ncpu <= 1 || gomaxprocs <= int32(sched.npidle+sched.nmspinning)+1 {
		return false
	}
	if p := getg().m.p.ptr(); !runqempty(p) {
		return false
	}
	return true
}
```



- **不能为单核CPU**：不然会出现一个goroutine在自旋，另一个goroutine持有锁但是没有CPU时间片的情况，自旋也便失去了意义；
- **不可以`GOMAXPROCS=1`或`GOMAXPROCS>1`但只有一个P在运行**：P是对处理器资源的抽象，所以和单核场景类似；
- **P本地的G队列需要为空**：不然相较于自旋，切换本地G执行更有效率。

### 饥饿模式

新来的goroutine不会进行自旋，直接排到队尾
