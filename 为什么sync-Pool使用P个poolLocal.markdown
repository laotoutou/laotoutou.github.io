---
title: 为什么sync.Pool使用P个poolLocal
date: 2021-01-17 03:02:44
tags: 
- golang
- src
---

看公司项目的日志模块有用到sync.Pool, 觉得有优化的点, 顺便看了看源码, 一边看一边给自己提问题：为什么sync.Pool使用P个poolLocal?

首先看下sync.Pool的用法, 其实对外暴露的就2个方法, Put(), Get()

Get()用于从Pool中获取对象, 并且一般在用完后Put()还回来, 否则下次Get()时会重新创建对象, 失去了Pool存在的价值
Pool.New()方法是用户定义的, 用来给sync.Pool增加元素(扩容),

看下sync.Pool结构:

```
type Pool struct {
	noCopy noCopy

	local     unsafe.Pointer // local fixed-size per-P pool,  actual type is [P]poolLocal
	localSize uintptr        // size of the local array

	victim     unsafe.Pointer // local from previous cycle
	victimSize uintptr        // size of victims array

	// New optionally specifies a function to generate
	// a value when Get would otherwise return nil.
	// It may not be changed concurrently with calls to Get.
	New func() interface{}
}
```

看代码注释, local是一个P大小的poolLocal结构:

```
type poolLocal struct {
	poolLocalInternal

	// Prevents false sharing on widespread platforms with
	// 128 mod (cache line size) = 0 .
	pad [128 - unsafe.Sizeof(poolLocalInternal{})%128]byte
}

// Local per-P Pool appendix.
type poolLocalInternal struct {
	private interface{} // Can be used only by the respective P.
	shared  poolChain   // Local P can pushHead/popHead; any P can popTail.
}
```

pad是用来解决cpu cache伪共享问题, 32位操作系统的cacheline是64字节, 64位操作系统cacheline 128字节
而解决伪共享问题的办法是pad, 避免多个变量共享一个cacheline

真正存放数据用来重用的位置是poolLocalInternal, 看注释其中private结构分别由各自的P使用, 而shard是共享的, 和GPM模型中P的本地goroutine队列和全局goroutine队列有点像(果然天下没有新东西)

**private只允许对应的P使用, 再联想GPM模型, 不论有多少个goroutine, cpu一个核心同时只能执行1个goroutine, go程序的同时并行数是P, 不难想到[P]poolLocal的作用是减少锁的争用, 如果sync.Pool只是简单使用链表存储, 那么每次Get()是必然需要加锁, 影响性能**

看源码验证下:

```
func (p *Pool) Get() interface{} {
	if race.Enabled {
		race.Disable()
	}
	l, pid := p.pin()
	x := l.private
	l.private = nil
	if x == nil {
		// Try to pop the head of the local shard. We prefer
		// the head over the tail for temporal locality of
		// reuse.
		x, _ = l.shared.popHead()
		if x == nil {
			x = p.getSlow(pid)
		}
	}
	runtime_procUnpin()
	if race.Enabled {
		race.Enable()
		if x != nil {
			race.Acquire(poolRaceAddr(x))
		}
	}
	if x == nil && p.New != nil {
		x = p.New()
	}
	return x
}
```

判断private=nil才取shard数据 

*p.pin()并没有加锁, 只是防止当前goroutine调离P*

在看shard取数据的操作:

```
func (d *poolDequeue) popHead() (interface{}, bool) {
	var slot *eface
	for {
		ptrs := atomic.LoadUint64(&d.headTail)
		head, tail := d.unpack(ptrs)
		if tail == head {
			// Queue is empty.
			return nil, false
		}

		// Confirm tail and decrement head. We do this before
		// reading the value to take back ownership of this
		// slot.
		head--
		ptrs2 := d.pack(head, tail)
		if atomic.CompareAndSwapUint64(&d.headTail, ptrs, ptrs2) {
			// We successfully took back slot.
			slot = &d.vals[head&uint32(len(d.vals)-1)]
			break
		}
	}

	val := *(*interface{})(unsafe.Pointer(slot))
	if val == dequeueNil(nil) {
		val = nil
	}
	// Zero the slot. Unlike popTail, this isn't racing with
	// pushHead, so we don't need to be careful here.
	*slot = eface{}
	return val, true
}
```

从shard取数据时使用了atomic.CompareAndSwapUint64自旋锁
