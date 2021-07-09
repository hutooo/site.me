---
title: RedisDIY - 简单动态字符串[SDS]
author: ash
tags: ["Redis", "NoSQL"]
categories: ["存储探秘", "Hacking", "DIY"]
date: 2021-04-12T19:46:47+08:00
image: 'miku01.jpg'
---


# 0x01 Redis的简单动态字符串 SDS

```c
struct sdshdr {
    int len;
    int free;
    char buf[];
}
```

# 0x02 Go的实现

```go
type sds struct {
    buf []byte
    len int
    free int
}
```

由于 Go 中内建的 string 类型可以较好的替代 SDS的工作，因此可以直接选用 string.