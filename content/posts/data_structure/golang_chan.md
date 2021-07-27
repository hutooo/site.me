---
title: Golang数据结构 --- channel/chan
author: ash
tags: ["DataStructure", "Golang", "channel/chan"]
categories: ["数据结构"]
date: 2020-05-01T10:30:00+08:00
cover: "/images/vr.jpg"
---

# 0x00 什么是channel/chan

channel是Go在语言层面提供的goroutine间的通信结构. 它非常轻便，也主要用于进程内多goroutine间的通信.

# 0x01 chan 数据结构

src/runtime/chan.go:hchan 对channel进行了定义:

```go
type hchan struct {
    qcount uint     // 当前队列剩余的元素个数
    dataqsiz uint   // 环形队列的长度，也即可存放的元素个数
    buf unsafe.Pointer // 环形队列指针
    elemsize uint16  // 每个元素的大小
    closed uint32  // 标识关闭状态
    elemtype *_type  // 元素类型
    sendx uint   // 队列下标，指示元素写入时存放到队列中的位置
    recvx uint   // 队列下标，指示元素从队列该位置读出
    recvq waitq  // 等待读消息的goroutine队列
    sendq waitq  // 等待写消息的goroutine队列
    lock mutex   // 互斥锁，chan不允许并发读写
}
```

## 环形队列

chan内部实现了一个环形队列作为缓冲，队列的长度是创建chan的时候指定的.

比如一个可缓存6元素的channel:

```s
|  hchan   |
------------           -------------------------
|qcount=2  |      |--> | 0 | 1 | 1 | 0 | 0 | 0 |
------------      |    -------------------------
|dataqsiz=6|      |          ↑       ↑
------------      |          |       |
|   buf    | -----|          |       |
------------                 |       |
|  sendx=3 | ------------------------|
------------                 |
|  recvx=1 | ----------------|
```

1. dataqsiz 表明队列长度为6
2. buf 指向队列的内存
3. qcount 表明队列中还有两个元素
4. sendx 指示后续写入的数据存储的位置，范围[0,6)
5. recvx 指示从该位置读取数据，范围[0,6)

## 等待队列

1. 从chan读数据，如果缓冲区为空或者没有缓冲区，则当前goroutine会被阻塞.
2. 向chan写数据，如果缓冲区已满或者没有缓冲区，则当前goroutine会被阻塞.

被阻塞的goroutine将会挂在channel的等待队列中:

    * 因读而阻塞的goroutine会被向channel写入数据的goroutine唤醒.
    * 因写而阻塞的goroutine会被从channel读取数据的goroutine唤醒.

看一个没有缓冲区的chan，且挂着几个等待读的goroutine:

```s
|  hchan   |
------------
| qcount=0 |
------------
|dataqsiz=0|
------------
|   buf    |
------------ 
|  sendx=0 |
------------
|  recvx=0 |
------------
|  recvq   | --> |G| --> |G| --> |G|
------------
|  sendq   |
```

> 通常 recvq 和 sendq 至少有一个为空.  只有当同一个goroutine使用select语句对chan同时读写数据时，是一个例外.

## 类型信息


