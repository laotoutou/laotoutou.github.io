---
title: sync.WaitGroup源码阅读
date: 2021-02-05 23:29:00
tags:
- golang
- src
---

WaitGroup用于golang的goroutine同步, 主goroutine等待子goroutine执行结束.

在主goroutine中对计数器进行 **Add()** 操作, 与之相对的, 子goroutine执行结束时执行 **Done()**, 对计数器Add(-1), 同时主goroutine执行 **Wait()** 操作, 等待计数器为0

总结就是3个方法, Add(), Done(), Wait()

```
// A WaitGroup waits for a collection of goroutines to finish.
// The main goroutine calls Add to set the number of
// goroutines to wait for. Then each of the goroutines
// runs and calls Done when finished. At the same time,
// Wait can be used to block until all goroutines have finished.
//
// A WaitGroup must not be copied after first use.
type WaitGroup struct {
	noCopy noCopy

	// 64-bit value: high 32 bits are counter, low 32 bits are waiter count.
	// 64-bit atomic operations require 64-bit alignment, but 32-bit
	// compilers do not ensure it. So we allocate 12 bytes and then use
	// the aligned 8 bytes in them as state, and the other 4 as storage
	// for the sema.
	state1 [3]uint32
}
```

作者开局亮大招, 这个2个字段的结构体涉及知识点: 内存对齐, 原子操作, golang禁止拷贝机制

### Tip1 nocopy
为什么WaitGroup不允许被复制, [https://bronzesword.medium.com/what-does-nocopy-after-first-use-mean-in-golang-and-how-12396c31de47](https://bronzesword.medium.com/what-does-nocopy-after-first-use-mean-in-golang-and-how-12396c31de47)

> Most of the time it is required so for safety reasons, for example you have a struct with a pointer field and you don’t want it to be copied since a shallow copy will make these two hold the same pointer and be unsafe.

翻译一下是由于安全原因, 假如另一个指针指向了WaitGroup.state1的地址将是不安全的, 因为这个数组保存着计数器的值

在golang中可以通过定义一个结构体, 并实现Lock()和Unlock()方法即可实现禁止拷贝, 而空结构体struct{}是0字节, 嵌套在struct中不会占用空间

```
// noCopy may be embedded into structs which must not be copied
// after the first use.
//
// See https://golang.org/issues/8005#issuecomment-190753527
// for details.
type noCopy struct{}

// Lock is a no-op used by -copylocks checker from `go vet`.
func (*noCopy) Lock()   {}
func (*noCopy) Unlock() {}
```

### Tip2 内存对齐

另外对于golang空结构体struct{}嵌套, 需要注意的是不要放在最后一位.

因为struct{}不占用空间, 放在最后一位时, golang为了避免其他指针指向这个空结构体而导致指针指向了结构体之外的内存地址, golang会自动为struct{}内存对齐, 浪费了空间. 这里WaitGroup将struct{}放在了第一个位置

继续看源码, WaitGroup中设计最精妙的地方
```
// state returns pointers to the state and sema fields stored within wg.state1.
func (wg *WaitGroup) state() (statep *uint64, semap *uint32) {
	if uintptr(unsafe.Pointer(&wg.state1))%8 == 0 {
		return (*uint64)(unsafe.Pointer(&wg.state1)), &wg.state1[2]
	} else {
		return (*uint64)(unsafe.Pointer(&wg.state1[1])), &wg.state1[0]
	}
}
```
state()会被Add()和Wait()调用, 用于获取两个变量statep和semap

开始灵魂发问

#### 怎么理解这个statep和semap?

* statep是64位变量, 而这个变量保存了2个数据: counter和waiter. 高32位保存counter, 低32位保存waiter. 具体怎么做的看Add方法
* semap为信号量, 执行wait的goroutine通过信号量semap阻塞(runtime_Semacquire), 等待被被子goroutine通过信号量semap唤醒(runtime_Semrelease)

#### 为什么把waiter和counter打包

可以看下sync.WaitGroup的提交历史, 在之前的某个版本里面, waiter、counter以及semap确实是分开的, 但是却多了一个互斥锁, 而把waiter和counter打包可以通过原子操作代替锁. 关于这个改动的review: [https://go-review.googlesource.com/c/go/+/4117/](https://go-review.googlesource.com/c/go/+/4117/)

#### 为什么要内存对齐?

由于内存的布局, 操作系统访问内存不是按变量直接访问, 而是按照字word获取内存数据, 一次内存访问获取一个word, 比如32位架构的机器一次只能寻址32位, 也就是4字节, 所以32位机器的字长word为4字节, 而64位架构机器的字长word为8字节

也就是1次内存访问可以获取1个word的数据, 假设一个8字节的int64变量没有按word内存对齐, 那么需要经历2次内存访问, 然后拼接起来才是完整的数据

当然不是一定要做内存对齐, 比如redis为了减少内存浪费, 有些结构是禁止内存对齐的

总结：内存对齐是为了提高数据访问效率, 提升性能

#### statep为什么要保证8字节对齐

[https://golang.org/pkg/sync/atomic/#pkg-note-BUG](https://golang.org/pkg/sync/atomic/#pkg-note-BUG)

64位机器可以一次访问内存的64位数据, 而32位机器访问64位数据需要两次内存访问.

> On ARM, x86-32, and 32-bit MIPS, it is the caller's responsibility to arrange for 64-bit alignment of 64-bit words accessed atomically. The first word in a variable or in an allocated struct, array, or slice can be relied upon to be 64-bit aligned.

翻译一下就是在32位机器上想要实现64位变量原子操作, 需要调用方保证变量是64位内存对齐的. (这是atomic包的要求)

具体怎么实现的打个TODO

#### 如何在不同架构(32/64)的机器上实现8字节内存对齐?

这里的要求的内存对齐是说变量地址是8字节对齐的, 即地址%8=0

32位架构的机器本身可以保证是4字节对齐的, 而64位架构是8字节对齐.

uint32是4字节, [3]uint32是12字节, 需要存放8字节的计数器和4字节的信号量, 这里为了原子操作计数器, 需要保证 计数器地址%8=0

```
if uint32数组地址%8==0:
	// 如果把前2个uint32作为计数器, 可以保证8字节对齐.
	// 但如果后两个uint32作为计数器, 那么计数器会被分成2个4字节
else:
	// 4字节对齐
	// 如果要让计数器8字节对齐, 需要用一个uint32来补齐前面的4字节
```

## Add

继续看Add()方法
```
// Add adds delta, which may be negative, to the WaitGroup counter.
// If the counter becomes zero, all goroutines blocked on Wait are released.
// If the counter goes negative, Add panics.
//
// Note that calls with a positive delta that occur when the counter is zero
// must happen before a Wait. Calls with a negative delta, or calls with a
// positive delta that start when the counter is greater than zero, may happen
// at any time.
// Typically this means the calls to Add should execute before the statement
// creating the goroutine or other event to be waited for.
// If a WaitGroup is reused to wait for several independent sets of events,
// new Add calls must happen after all previous Wait calls have returned.
// See the WaitGroup example.
```
翻译一下源码的注释: Add方法接收一个参数delta, 把delta添加到上文提到的计数器counter中, 也就是那个64位变量中, 并且delta可以为负数, 但计数器counter不能为负数, 否则主动panic, 当counter从0再变到0时, 所有被Wait方法阻塞的goroutine将被唤醒.


```
func (wg *WaitGroup) Add(delta int) {
	// 调用state方法获取statep(counter和waiter) 和信号量semap
	statep, semap := wg.state()

	// ...

	// statep分2部分, 高32位存储计数器counter, 将delta加到计数器中
	state := atomic.AddUint64(statep, uint64(delta)<<32)
	// state右移32位可以取出计数器counter
	v := int32(state >> 32)
	// 注意state是uint64, 强制转化位uint32可以取低32位, 也就是waiter
	w := uint32(state)

	// ...

	// 计数器只能从0加上若干个delta后再变回0, 但是不能小于0
	if v < 0 {
		panic("sync: negative WaitGroup counter")
	}

	// 给计数器加delta, 但是waiter!=0
	// waiter!=0说明已经调用了Wait方法(获得cas锁后会给waiter+1), 这时却尝试增加计数器的值, 这是不被允许的
	// 也就是说调用Wait方法被调用后, 只能对计数器进行减操作(通过Done或者 Add负数)
	if w != 0 && delta > 0 && v == int32(delta) {
		panic("sync: WaitGroup misuse: Add called concurrently with Wait")
	}

	// 在这里return说明是需要被wait的goroutine开始时的调用(goroutine开始时Add(1), 结束后Done())
	if v > 0 || w == 0 {
		return
	}

	// 程序运行到这里说明调用了Wait之后, 被等待的goroutine内部调用Done或者 Add负数,
	// 且计数器counter变回了0
	// 所以这里要进行收尾工作: 重置counter和waiter, 唤醒因Wait被阻塞的goroutine

	// This goroutine has set counter to 0 when waiters > 0.
	// Now there can't be concurrent mutations of state:
	// - Adds must not happen concurrently with Wait,
	// - Wait does not increment waiters if it sees counter == 0.
	// Still do a cheap sanity check to detect WaitGroup misuse.

	// Add方法没有加锁, 上文使用了atomic原子操作增加statep指针指向的值
	// 但这里*statep!=state说明有其他地方修改了*statep, 这是不被允许的
	if *statep != state {
		panic("sync: WaitGroup misuse: Add called concurrently with Wait")
	}

	// Reset waiters count to 0.
	// 重置counter和waiter
	*statep = 0

	// runtime_Semrelease通过信号量semap唤醒被阻塞的goroutine
	// 与之相对的是Wait方法中的runtime_Semacquire
	for ; w != 0; w-- {
		runtime_Semrelease(semap, false, 0)
	}
}

// Done decrements the WaitGroup counter by one.
func (wg *WaitGroup) Done() {
	wg.Add(-1)
}
```

## Wait

Wait方法会被阻塞, 直到计数器counter变为0. (Add方法会保证中间不会变成负数, 否则主动panic)

```
// Wait blocks until the WaitGroup counter is zero.
func (wg *WaitGroup) Wait() {
	// 获取statep 和信号量semap
	statep, semap := wg.state()
	if race.Enabled {
		_ = *statep // trigger nil deref early
		race.Disable()
	}
	for {
		state := atomic.LoadUint64(statep)
		// 右移32获取state的高32位的counter
		v := int32(state >> 32)
		// 强制转化位uint32, 取waiter
		w := uint32(state)

		// 计数器counter=0, 直接return不用阻塞
		if v == 0 {
			// Counter is 0, no need to wait.
			if race.Enabled {
				race.Enable()
				race.Acquire(unsafe.Pointer(wg))
			}
			return
		}

		// Increment waiters count.
		// 获取cas锁后为statep+1, 也就是为低32位的waiter+1
		if atomic.CompareAndSwapUint64(statep, state, state+1) {
			// ...

			// 被阻塞
			runtime_Semacquire(semap)

			// 程序运行到这里说明goroutine被 执行Add的runtime_Semrelease的goroutine 唤醒了
			// 但是如果*statep!=0 说明其他地方修改了statep指向的值
			if *statep != 0 {
				panic("sync: WaitGroup is reused before previous Wait has returned")
			}

			// ...

			return
		}

		// 没获得cas锁, 继续循环
	}
}
```

参考

[What does “nocopy after first use” mean in golang and how](https://bronzesword.medium.com/what-does-nocopy-after-first-use-mean-in-golang-and-how-12396c31de47)

[内存布局](https://gfw.go101.org/article/memory-layout.html)

[Go中由WaitGroup引发对内存对齐思考](https://www.luozhiyun.com/archives/429)

[sync.WaitGroup将waiter和counter打包的提交历史](https://go-review.googlesource.com/c/go/+/4117/)
