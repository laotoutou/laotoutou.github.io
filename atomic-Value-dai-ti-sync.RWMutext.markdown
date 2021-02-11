---
title: atomic.Value代替sync.RWMutex
date: 2021-01-16 02:00:29
tags: 
- golang
- src
---

记一次性能优化，读公司项目代码时候，发现好些使用sync.RWMutext的使用场景：项目启动时候对高频数据缓存到内存缓存中，同时每隔一段时间重新写一下这个缓存（用一个全局变量）：
```
type cosCred struct {
	Cred []int64
	sync.RWMutex
}

var CosCred *cosCred

// 每分钟写一次
func InitCosCred() {
	CosCred = new(cosCred)
	CosCred.Cred, _ = GetGlobalCredData()
	go func() {
		for range time.NewTicker(10 * time.Minute).C {
			cred, err := GetGlobalCredData()
			if err != nil {
				continue
			}
			CosCred.Lock()
			CosCred.Cred = cred
			CosCred.Unlock()
		}
	}()
}

// 接口会并发读
func GetCosCred() []int64 {
	CosCred.RLock()
	defer CosCred.RUnlock()
	if CosCred.Cred == nil {
		return nil
	}
	resp := CosCred.Cred
	return resp
}
```

看到以上场景是一个协程写，接口并发读，而写的过程是直接修改变量的值(切片引用类型，修改了全局变量指向的底层数组)，只是这一个写的过程，却要在每次读的时候加读锁，其实只要换成保证原子操作的变量赋值和读取就行了，使用atomic.Value

atomic.Value封装了一个interface{}类型的v变量，简单粗暴，同时提供了Load和Store原子操作，用于给v原子赋值和读取，简单修改下：
```

var CosCred atomic.Value

// 每分钟写一次
func InitCosCred() {
	data, _ := GetGlobalCredData()
	CosCred.Store(data)
	go func() {
		for range time.NewTicker(10 * time.Minute).C {
			cred, err := GetGlobalCredData()
			if err != nil {
				continue
			}
			CosCred.Store(data)
		}
	}()
}

// 接口会并发读
func GetCosCred() []int64 {
	resp, _ := CosCred.Load().([]int64)
	return resp
}
```

*原子操作由底层硬件支持，而锁则由操作系统的调度器实现。锁应当用来保护一段逻辑，对于一个变量更新的保护，原子操作通常会更有效率*

关于atomic.Value有一篇不错的blog [https://blog.betacat.io/post/golang-atomic-value-exploration/](https://blog.betacat.io/post/golang-atomic-value-exploration/)

