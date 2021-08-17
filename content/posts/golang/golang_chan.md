---
title: Golang数据结构 --- channel/chan
author: ash
tags: ["Golang", "channel/chan"]
categories: ["Golang剖析"]
date: 2020-05-05T21:30:00+08:00
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

一个chan只能传递一种类型的数据，类型信息存储在hchan数据结构中:

* elemtype 代表类型，用于数据传递过程中的赋值.
* elemsize 代表类型大小，用于在buf中定位元素的位置.

## 锁

一个chan仅允许同时被一个goroutine读写.

# 0x02 chan 读写

## chan的创建

chan的创建过程其实就是 hchan结构的初始化过程. 其中类型信息和缓冲区长度由make语句传入，buf的大小由元素类型大小和缓冲区长度共同决定.

实例伪代码:

```go
func makechan(T *chantype, size int) *hchan {
    var c *hchan
    c = new(hchan)
    c.buf = malloc(sizeof(T) * size)
    c.elemsize = sizeof(T)
    c.elemtype = T
    c.dataqsiz = size
    return c
}
```

## 向chan写入数据

1. 如果等待接收队列recvq不为空，说明缓冲区中没有数据或者没有缓冲区. 此时直接从recvq中取出G，将数据写入，最后唤醒该G，结束发送过程.
2. 如果缓冲区中有空余的位置，将数据写入缓冲区，结束发送过程.
3. 如果缓冲区中没有空余位置，将待发送数据写入G，将G加入当前的sendq，进入睡眠，等待被读goroutine唤醒.

## 从chan读取数据

1. 如果等待发送的队列sendq不为空，且没有缓冲区，直接从sendq中取出G，读出G中数据，最后把G唤醒，结束读取过程.
2. 如果等待发送队列sendq不为空，说明缓冲区已满，从缓冲区首部读取数据，然后从sendq中取出一个G，把G中的数据写入buf队尾，将G唤醒，结束读取过程.
3. 如果缓冲区中有数据，从缓冲区读取数据，结束读取过程.
4. 将当前的goroutine加入recvq队列，进入睡眠，等待被写goroutine唤醒.

## 关闭 chan

关闭chan，会把recvq中的G全部唤醒，本该写入G的数据置为nil，把sendq中的G全部唤醒，但是这些G会引发panic.

此外，

* 关闭值为nil的chan
* 关闭已经关闭的chan
* 向已经关闭的chan写入数据

都会引发panic.

# 0x03 常用示例

## 单向chan

单向chan指的就是只用于接收或者只用于写入的chan. 

> 实际并没有真正的单向chan. 只是对chan的一种使用限制，类似于C语言的const修饰.

* func readChan(ch <-chan int)  // 通过形参限定函数内只能从chan中读取数据
* func writeChan(ch chan<- int) // 通过形参限定函数内只能向chan中写入数据

举个栗子:

```go
func readChan(ch <- chan int) {
    <-ch
}

func writeChan(ch chan<- int) {
    ch<-1
}

func main() {
    var mychan = make(chan int, 10) // 正常channel
    writeChan(mychan)  // 只写限制
    readChan(mychan)   // 只读限制
}
```

## select语句

select语句用于监控channel. 

举栗:

```go
func addNumToChan(ch chan int) {
    for {
        ch <- 1
        time.Sleep(1 * time.Second)
    }
}

func main() {
    var chan1 = make(chan int, 10)
    var chan2 = make(chan int, 10)

    go addNumToChan(chan1)
    go addNumToChan(chan2)

    for {
        select {
        case v := <-chan1:
            fmt.Printf("Get elem %d from chhan1.\n", v)
        case v := <-chan2:
            fmt.Printf("Get elem %d from chhan2.\n", v)
        default:
            fmt.Printf("No elem in chan1 or chan2.\n")
            time.Sleep(1 * time.Second)
        }
    }
}
```

程序中创建了两个channel: chan1 和 chan2. 函数 addNumToChan 向两个channel中周期性写入数据. 通过select监控两个chan，任意一个可读时就从其中读出数据.

```sh
# 程序输出
D:\SourceCode\GoExpert\src>go run main.go
Get elem 1 from chan1.
Get elem 1 from chan2.
No elem in chan1 or chan2.
Get elem 1 from chan2.
Get elem 1 from chan1.
No elem in chan1 or chan2.
Get elem 1 from chan2.
Get elem 1 from chan1.
No elem in chan1 or chan2.
```

通过输出可以发现，从channel中读出数据的顺序是随机的. [select语句的多个case执行顺序是随机的]

select的case语句读channel不会阻塞，尽管channel中没有数据. 这是由于case语句编译后调用读channel时会明确传入不阻塞的参数，此时读不到数据时不会将当前goroutine加入到等待队列，而是直接返回.

## range 语句

使用range语句可以持续地从channel中读出数据，好像在遍历数组一样，当channel中没有数据时会阻塞当前goroutine，与读channel时阻塞处理机制相同.

```go
func chanRange(ch chan int) {
    for e := range ch {
        fmt.Printf("Get elem %d from chan.\n", e)
    }
}
```

> 向这个chan写入数据的goroutine退出时，系统会检测到这种情况后并panic，否则range将永久阻塞.

