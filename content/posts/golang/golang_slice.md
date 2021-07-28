---
title: Golang数据结构 --- slice切片
author: ash
tags: ["DataStructure", "Golang", "slice"]
categories: ["Golang剖析"]
date: 2020-05-06T21:30:00+08:00
cover: "/images/vr.jpg"
---

# 0x00 什么是slice/切片

slice也称动态数组，底层依托于数组，可以实现扩容、传递...  使用上较数组更为灵活.

也因为灵活，所以必须要熟练掌握原理，才不会轻易掉入slice陷阱.

# 0x01 slice的实现原理

slice的实现依托数组(数组会对用户屏蔽)，在容量不足时能自动扩容.

## slice 数据结构

slice结构在 src/runtime/slice.go:slice  中进行了定义:

```go
type slice struct {
    // 指向底层数组
    array unsafe.Pointer
    // 长度
    len int
    // 容量
    cap int
}
```

## slice 的创建

1. 使用make语句构建slice，构建时可以传入长度和容量，底层数组的长度等于容量.

栗子:

```go
slice := make([]int, 5, 10)
```

其结构示意图如下:

```s
| slice |
---------         {---len---}
| array | ------> |0|0|0|0|0|0|0|0|0|0|
---------         {-------- cap ------}
| len=5 |
---------
|cap=10 |
```

2. 使用数组创建slice, slice将与原数组公用一部分内存.

栗子:

```go
array := [10]int{0,1,2,3,4,5,6,7,8,9}
slice := array[5:7]
```

其结构示意图如下:

```s
| slice |
---------                   {len}
| array | ------> |0|1|2|3|4|5|6|7|8|9|
---------                   {---cap---}
| len=2 |
---------
| cap=5 |
```

切片从数组array[5]开始，到array[7]结束(不包含array[7]). 因此切片长度为2，而5开始之后所有内容都作为切片的预留内存，因此容量时5.

> 数组和切片操作可能作用在同一块内存!!

## slice 扩容


