---
title: snowflake-go 阅读
date: 2021-01-16 02:18:55
tags: src
---

github地址：[https://github.com/bwmarrin/snowflake](https://github.com/bwmarrin/snowflake)

总共就300行代码，主要逻辑也就100行吧。

这是一款大数据量下的id生成器的snowflake-golang实现，snowflake生成的id (int64类型)包含毫秒时间戳、机器id、同一毫秒下的自增id这3部分数据，这里面主要是位运算的妙用(好多开源项目都会用到位运算)

用int64的64bit存储以下部分：

* 12bit的自增id **step**(同一毫秒下)
* 10bit的机器id **node**(多台机器)
* 41bit的毫秒时间间距 **time**(不用从1970开始算)
* 1bit的unset 预留

这几个参数,12bit 10bit 41bit 1bit其实都可以根据自己情况自定义：

* 41bit的毫秒**time**最多可以表示``` ((1<<41)-1) / (86400*1000*465) = 69.7```年(减1是因为包括0)
* 10bit的**node**最多可表示```(1<<10)-1=1023```个机器
* 12bit的**step**同一毫秒最多可表示```(1<<12)-1=4095```个自增id (同一机器同一毫秒生产的id数目大于4095怎么办，代码就体现了)

看看怎么用：
```
	// Create a new Node with a Node number of 1
	node, err := snowflake.NewNode(1)
	if err != nil {
		fmt.Println(err)
		return
	}

	// Generate a snowflake ID.
	id := node.Generate()

	// Print out the ID's timestamp
	fmt.Printf("ID Time  : %d\n", id.Time())

	// Print out the ID's node number
	fmt.Printf("ID Node  : %d\n", id.Node())

	// Print out the ID's sequence number
	fmt.Printf("ID Step  : %d\n", id.Step())
```

## 先来生成id

不过生成id前,先new一个node对象
```
func NewNode(node int64) (*Node, error) {

	n := Node{}
	n.node = node
	n.nodeMax = -1 ^ (-1 << NodeBits) // == (1<< NodeBits) - 1
	n.nodeMask = n.nodeMax << StepBits // nodeMask以及stepMask主要用来由生成的id反推node和step
	n.stepMask = -1 ^ (-1 << StepBits) // == (1<< StepBits) - 1
	n.timeShift = NodeBits + StepBits // time右移NodeBits + StepBits才是id中time对应的位置
	n.nodeShift = StepBits // node右移StepBits才是id中node对应的位置
	// step是从0位到12位

	if n.node < 0 || n.node > n.nodeMax {
		return nil, errors.New("Node number must be between 0 and " + strconv.FormatInt(n.nodeMax, 10))
	}

	var curTime = time.Now()
	// add time.Duration to curTime to make sure we use the monotonic clock if available
	n.epoch = curTime.Add(time.Unix(Epoch/1000, (Epoch%1000)*1000000).Sub(curTime))

	return &n, nil
}
```

以上的StepBits, NodeBits, Epoch都是配置项

* StepBits = 12
* NodeBits = 10
* Epoch = 1288834974657 (毫秒时间戳，这里表示的是2010年)

注意Epoch不用从1970开始算，总共才有41bit表示毫秒时间，从1970开始有点浪费，可以设置为距项目上线时间最近的时间，可以持续69年生成id。如果从1970年开始算，41bit还可以存储18年的毫秒时间

正式生成id
```
func (n *Node) Generate() ID {

	n.mu.Lock()

	// nanoseconds 1e9
	// now单位毫秒
	now := time.Since(n.epoch).Nanoseconds() / 1000000

	// 每毫秒可以产生n.stepMask个id
	// n.step的值[0, n.stepMask]
	if now == n.time {
		n.step = (n.step + 1) & n.stepMask

		// 当1毫秒产生的id个数大于n.stepMask时
		if n.step == 0 {
			// 强制sleep直到下一毫秒
			for now <= n.time {
				now = time.Since(n.epoch).Nanoseconds() / 1000000
			}
		}
	} else {
		// 当前这一毫秒还没有生成id,用0即可
		n.step = 0
	}

	n.time = now

	// 所以r由3部分组成: time node step
	// shift表示位移量
	// 或操作 只要对应位有1个为1就为1，方便由r反推time node step
	r := ID((now)<<n.timeShift |
		(n.node << n.nodeShift) |
		(n.step),
	)

	n.mu.Unlock()
	return r
}
```

所以上面的问题，同一台机器1ms生成的id数大于4095就是死循环直到下一个ms.

## 由id反推time node step

```
func (f ID) Time() int64 {
	return (int64(f) >> timeShift) + Epoch
}

func (f ID) Node() int64 {
	// 位运算优先级高
	return int64(f) & nodeMask >> nodeShift
}

func (f ID) Step() int64 {
	// f的后stepBits位为step
	// stepMask为step所占用的stepBits个位的最大值
	// 与运算结果的最大值为stepMask
	return int64(f) & stepMask
}
```

生成id时用的或运算，反推用与运算
