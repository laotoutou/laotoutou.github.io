---
title: lru并发get性能优化
date: 2021-02-25 22:16:00
tags:
- golang
- 锁
---

LRU(Least recently used)由一个字典和链表组成, 字典的作用是快速定位到目标元素, 但是只用字典不方便做淘汰机制, 因此需要一个链表维护各个元素的访问时间, 当元素总数超过指定大小时, 删除链表最后元素.

这导致每次Get操作需要将链表对应元素移动到链表头部, 而对于并发情况下的写操作, 需要用互斥原语保证操作的正确性, 互斥锁导致的问题是即使在多核情况下, 同一时刻只能有一个线程Get, 这必然影响并发性能

那么如何减小锁粒度, 这里有几种思路:

## 分片 

对key做hash, 根据hash值将key映射到不同的lru链表中. 这是一种比较常见的思路, 分片之后同一个lru的Get在同一时刻仍然是互斥的, 但是各个lru之间是可以并发访问的.

但是分片的数量不能过多, 否则不利于缓存淘汰. 考虑到cpu调度时同一时刻最大并行数为cpu核心数, 这里设置为cpu核心数即可.

但是需要选择合适的hash函数, 使key分布均匀且hash成本不大于获取互斥锁的成本

这里推荐使用times33哈希函数, 简单高效

## 无锁数据结构

无锁数据结构是通过硬件提供的原子操作指令代替操作系统系统的锁, 但是对于链表只能用于单链表元素的新增和删除.

lru中的链表需要是包含前后指针的链表, 原因是对于链表元素的移动, 需要获取前后结点, 因此对于lru无法使用无锁链表

## 互斥锁改读写锁

具体方式是创建lru时, 启动一个单线程/协程, 用来专门对链表做更新操作, 而对外的接口直接操作字典, 这种方式可以将原来的互斥锁改为读写锁.

golang的goroutine+channel很适合使用这种方式, 创建一个goroutine最少只用2k内存, 而goroutine之间使用channel进行同步

## 减小锁的次数

lru是在每次命中后将链表元素移动到链表头结点, 通过增加命中次数限制锁的次数, 比如设置命中3次才移动到头结点, 但是这种方式影响lru的准确性

# tslru做的优化:

[https://github.com/laotoutou/golang-lru](https://github.com/laotoutou/golang-lru)

这里使用了golang-lru的接口, 实现了tslru: thread safe lru. 在git仓库的tslru分支下可以找到源代码


## 单协程

创建tslru对象时, 启动一个协程从channel中获取数据进行更新/删除操作. get命中之后可以讲元素放到channel中异步操作

## 内存池

考虑到lru固定大小, 在达到固定大小时, 需要删除末尾元素, 之后的add操作需要创建一个新的元素, 可以使用内存池减小gc压力

## sync.Map代替原生map + sync.RWMutex

sync.Map的通过维护一个只读map, 从而提升并发读性能. 但是sync.Map没有对外提供获取map长度的方法,因此需要在外层维护一个size字段.

引入size字段后引入一个新的问题: 在取消互斥锁的情况下, 如何保证sync.Map和size字段原子更新? 

虽然没有了锁, 但是多了一个单线程, 可以将对sync.Map的更新操作放在单线程里面, 更新sync.Map后使用硬件提供的原子操作更新size

为什么要原子更新size, 直接更新不行吗? 

不行, 因为size需要对外提供获取长度的方法, 而高级语言的赋值在硬件层面并不是原子操作, 这里使用golang提供的atomic原子更新方法

使用以上几点可以显著增加lru的并发get性能, 但是add性能下降, 原因是add操作需要等待channel中的操作结束, 同时sync.Map的并发set性能也比原生map+锁的性能差, 因此通过第3点弥补add性能

## 分片

使用times33哈希函数对key做hash, 将不同的key映射到不同的lru中, 桶的个数为cpu核心数

### 不同hash函数性能对比

```
➜  go test src/testdir/hash_test.go -bench=.
goos: darwin
goarch: amd64
BenchmarkFnv-4     	25169342	        45.9 ns/op
BenchmarkSm-4      	26317040	        43.5 ns/op
BenchmarkCrc32-4   	23452971	        51.3 ns/op
BenchmarkMur-4     	86075402	        13.9 ns/op
BenchmarkDjb-4     	84505095	        14.3 ns/op
PASS
ok  	command-line-arguments	6.104s
```

因为使用相对简单,性能高的djb算法. 

测试代码:

```
import (
	"hash/crc32"
	"hash/fnv"
	"testing"

	"github.com/creachadair/cityhash"
	"github.com/huichen/murmur"
)

const (
	key  = "family:1234567"
	mask = 1023
)

func BenchmarkFnv(b *testing.B) {
	for i := 0; i < b.N; i++ {
		h := fnv.New32a()
		h.Write([]byte(key))
		_ = h.Sum32() & mask
	}
}

func BenchmarkSm(b *testing.B) {
	for i := 0; i < b.N; i++ {
		_ = cityhash.Hash32([]byte(key)) & mask
	}
}

func BenchmarkCrc32(b *testing.B) {
	for i := 0; i < b.N; i++ {
		_ = crc32.ChecksumIEEE([]byte(key))
	}
}

func BenchmarkMur(b *testing.B) {
	for i := 0; i < b.N; i++ {
		_ = murmur.Murmur3([]byte(key))
	}
}

/*
unsigned long
    hash(unsigned char *str)
    {
        unsigned long hash = 5381;
        int c;

        while (c = *str++)
            hash = ((hash << 5) + hash) + c; // hash * 33 + c

        return hash;
    }
*/

func BenchmarkDjb(b *testing.B) {
	for i := 0; i < b.N; i++ {
		_ = djb(key)
	}
}

func djb(key string) uint32 {
	var h rune = 5381
	for _, r := range key {
		h = ((h << 5) + h) + r
	}
	return uint32(h)
}
```

## 性能对比

这里用tslru与golang-lru, ccache做并发get性能对比:

测试标准是1s执行的get数, 4个协程并发执行

```
➜ go run src/testdir/main.go
glru  6239617
ts  14495863
cs  13136426
```

测试代码:

```
import (
	"fmt"
	"strconv"
	"sync/atomic"
	"time"

	lru2 "golang-lru"

	lru "github.com/hashicorp/golang-lru"
	"github.com/karlseguin/ccache"
)

func main() {
	glru()
	ts()
	clru()
}

func c(c *ccache.Cache) {
	t := time.After(time.Second)
	for i := 1; atomic.LoadInt32(&cstop) == 0; i++ {
		select {
		case <-t:
			atomic.SwapInt32(&cstop, 1)
			return
		default:
			_ = c.Get(strconv.Itoa(i & (i - 1)))
			//c.Add(strconv.Itoa(i), i)
			atomic.AddInt32(&cnum, 1)
		}
	}
}

func g(c *lru.Cache) {
	t := time.After(time.Second)
	for i := 1; atomic.LoadInt32(&gstop) == 0; i++ {
		select {
		case <-t:
			atomic.SwapInt32(&gstop, 1)
			return
		default:
			_, _ = c.Get(strconv.Itoa(i & (i - 1)))
			//c.Add(strconv.Itoa(i), i)
			atomic.AddInt32(&gnum, 1)
		}
	}
}

func t(c *lru2.TSCache) {
	t := time.After(time.Second)
	for i := 1; atomic.LoadInt32(&tstop) == 0; i++ {
		select {
		case <-t:
			atomic.SwapInt32(&tstop, 1)
			return
		default:
			_, _ = c.Get(strconv.Itoa(i & (i - 1)))
			//c.Add(strconv.Itoa(i), i)
			atomic.AddInt32(&tnum, 1)
		}
	}
}

var gnum, gstop int32

func glru() {
	c, err := lru.New(5)
	if err != nil {
		fmt.Println(err)
		return
	}

	c.Add("1", 1)
	c.Add("2", 2)
	c.Add("3", 3)
	c.Add("4", 4)
	c.Add("5", 5)

	go g(c)
	go g(c)
	go g(c)
	go g(c)

	time.Sleep(time.Second * 2)
	fmt.Println("glru ", gnum)
}

var tstop, tnum int32

func ts() {
	c, err := lru2.NewTSCache(5, 4)
	if err != nil {
		fmt.Println(err)
		return
	}

	c.Add("1", 1)
	c.Add("2", 2)
	c.Add("3", 3)
	c.Add("4", 4)
	c.Add("5", 5)

	go t(c)
	go t(c)
	go t(c)
	go t(c)
	time.Sleep(time.Second * 2)
	fmt.Println("ts ", tnum)
}

var cstop, cnum int32

func clru() {
	var cache = ccache.New(ccache.Configure())
	cache.Set("1", 1, time.Minute*10)
	cache.Set("2", 2, time.Minute*10)
	cache.Set("3", 3, time.Minute*10)
	cache.Set("4", 4, time.Minute*10)
	cache.Set("5", 5, time.Minute*10)

	go c(cache)
	go c(cache)
	go c(cache)
	go c(cache)
	time.Sleep(time.Second * 2)
	fmt.Println("cs ", cnum)
}
```
