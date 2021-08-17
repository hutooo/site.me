---
title: Golang数据结构 --- map
author: ash
tags: ["Golang", "map"]
categories: ["Golang剖析"]
date: 2020-05-07T20:15:00+08:00
cover: "/images/vr.jpg"
---

# 0x00 Golang 中的 map

Golang中的map使用散列表作为底层结构, 一个散列表中可以有多个散列节点[称为bucket], 每个bucket就保存了map中的一个或者一组键值对/映射.

# 0x01 map 的数据结构

map 结构在 src/runtime/map.go:hmap 中进行了定义:

```go
type hmap struct {
    count int // 当前保存的元素个数
    ...
    B     uint8 // 指示bucket数组的大小
    ...
    buckets unsafe.Pointer // buckets数组的指针，数组大小: 2^B
    ...
}
```

示栗图:

```s
|  hmap |
---------
| count |
---------
|  ...  |          -----------
---------     |--->| bucket0 |
|   B   |     |    -----------
---------     |    | bucket1 |
|  ...  |     |    -----------
---------     |    | bucket2 |
|buckets| ----|    -----------
---------          | bucket3 |
|  ...  |          -----------
```

示例图中，`hmap.B = 2`， 而`hmap.buckets` 长度是 2^B = 4. 元素经过哈希运算后会落到某个bucket中进行存储.

