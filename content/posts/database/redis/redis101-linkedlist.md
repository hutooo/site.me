---
title: RedisDIY - 链表
author: ash
tags: ["Redis", "NoSQL"]
categories: ["存储探秘", "Hacking", "DIY"]
date: 2021-04-13T15:28:17+08:00
cover: "/images/miku01.jpg"
---


# 0x01 Redis 链表

Redis链表结构

```c
typedef struct listNode {
    // 前置节点
    struct listNode *prev;

    // 后置节点
    struct listNode *next;

    // 节点的值
    void *value;
} 
```

多个 listNode 通过prev, next指针组成双端链表~

```s
          |listNode|          |listNode|          |listNode|
---next-->|--------|---next-->|--------|---next-->|--------|---next-->
          | value  |          | value  |          | value  |
<--prev---|  ...   |<--prev---|  ...   |<--prev---|  ...   |<--prev---
```

仅仅使用listNode已经可以组成链表，但使用统一的list结构持有链表则更加方便.

```c
typedef struct list {
    // 链表表头
    listNode *head;
    // 链表表尾
    listNode *tail;
    // 链表长度/包含的节点数量
    unsigned long len;
    // 节点值复制函数
    void*(*dup) (void *ptr);
    // 节点值释放函数
    void*(*free) (void *ptr);
    // 节点值对比函数
    int (*match) (void *ptr, void* key);
}
```

```s
| list |
|------|               |listNode|          |listNode|          |listNode|
| head |-------------->|--------|---next-->|--------|---next-->|--------|-->NULL
|      |               | value  |          | value  |          | value  |
| tail |        NULL<--|  ...   |<--prev---|  ...   |<--prev---|  ...   |
    |                                                         ↗
    |--------------------------------------------------------↗
| len 3|
| dup  |-->...
| free |-->...
| match|-->...
```

## Reids链表特性

1. 