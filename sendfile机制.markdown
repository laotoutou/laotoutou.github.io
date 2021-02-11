---
title: sendfile机制
date: 2021-01-30 18:24:19
tags: os
---

假如需要从本地磁盘或网卡读取数据发送给客户端
```
read(file, tmp_buf, len);
write(socket, tmp_buf, len);
```

1. read需要从磁盘拷贝数据到操作系统内核缓冲区
2. 内核缓冲区拷贝数据到进程空间
3. write从进程空间拷贝数据到内核缓冲区
4. 再从内核拷贝到发送网卡

经历4次拷贝过程

而sendfile从硬盘拷贝到内核, 再从内核拷贝到网卡缓冲区. 2次拷贝

为啥sendfile不能直接从磁盘拷贝到网卡?
我理解是因为DMA(Direct Memory Access)只是准备数据, 只是给cpu发送信号表示数据准备好了, 但是还是需要cpu去调度, 而不能主动发送

使用场景:
1. nginx的静态资源分发
2. kafka消费者读取数据: 从磁盘拷贝到内核缓冲区, 再从内核缓冲区到网卡缓冲区
