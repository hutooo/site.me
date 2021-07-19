---
title: An Empirical Evaluation of In-Memory Multi-Version Concurrency Control
author: ash
tags: ["MVCC", "DataBase", "Concurrency", "Paper"]
categories: ["阅读与感悟"]
date: 2021-04-21T14:46:02+08:00
cover: "/images/chino03.jpg"
---

## 0x00 摘要

多版本并发控制(MVCC)是目前 现代数据库管理系统(DBMS)中 最流行的事务管理方案. MVCC在20世纪70年代后期被提出，但是过去十年里，主流的关系型数据库都开始采用它. MVCC维护多个版本的数据，使得DMBS在处理事务时，不违背ACID的同时，增加并行性. 但是，在多核和内存设置中扩展 MVCC 并非易事: 当有大量线程并行运行时，同步的开销可能会超过MVCC带来的好处.

为了理解 MVCC 在现代硬件中处理事务时的表现，我们对4个关键设计决策进行了广泛的研究: 并发控制协议、版本存储、垃圾回收 和 索引管理. 我们在基于内存的 DBMS 中实现了所有这些的最新变体，并使用 OLTP 工作负载评估它们. 我们的分析确定了每个设计选择的基本瓶颈.

## 0x01 前言

计算机体系结构的进步导致了多核、基于内存的DBMS 的兴起，它们采用高效的事务管理机制来最大化并行性. 在DMBS中使用最广泛的方案是多版本并发控制协议(MVCC). MVCC 的基本思想是，DBMS 维护数据库中每个逻辑对象的多个物理版本，以允许对同一对象的操作并行进行. 这些对象可以是任何粒度的，但几乎每个 MVCC DBMS 都使用元组，因为它在并行性和版本跟踪之间提供了很好的平衡. 多版本控制允许只读事务访问元组的旧版本，而不会阻止读写事务同时生成新版本. 在单一版本的系统中，事务总是使用新的信息编写元组来更新它.

本文对 MVCC 数据库管理系统中的关键事务管理设计决策进行了研究: 
    
    1. 并发控制协议
    2. 版本存储
    3. 垃圾回收
    4. 索引管理

对于每个主题，我们描述内存中 DBMS 的最新实现，并讨论它们的优缺点. 我们还强调了阻止它们伸缩以支持更大的线程数和更复杂的工作负载的问题. 我们的分析确定了强调实现的场景，并讨论了缓解这些场景的方法(如果可能的话).

## 0x02 背景

我们首先概述 mvcc 的高级概念，然后讨论 DBMS 用于跟踪传输和维护版本控制信息的元数据.

### 2-1 MVCC 总览

事务管理方案允许终端用户以多线程的方式访问数据库，同时保持每个数据库都在专用系统上单独执行的假象. 它保证了DBMS的原子性和隔离性.

多版本系统对于现代数据库应用程序有几个优点. 最重要的是，它可能比单版本系统允许更大的并发性. 例如，MVCC DBMS 允许事务读取一个对象的旧版本，同时另一个事务更新同一个对象. 这一点很重要，因为在数据库上执行只读查询的同时，读写事务会继续更新它. 如果 DBMS 从未删除旧版本，那么系统还可以支持“时间旅行”操作，允许应用程序查询数据库在过去某个时间点存在的一致快照.

这些优点使 MVCC 成为近些年的新型数据库管理系统中最受欢迎的选择. 

下表提供了过去三十年中 MVCC 实现的总体情况:

|DBMS|年份|并发控制协议|版本存储|垃圾回收|索引管理|
|---|---|---|---|---|---|
|Oracle|1984|MV2PL|Delta|Tuple-Level(VAC)|Logical Pointers(Tupled)|
|Postgres|1985|MV2PL/SSI|Append-Only(O2N)|Tuple-Level(VAC)|Physical Pointers|
|MySQL-InnoDB|2001|MV2PL|Delta|Tuple-Level(VAC)|Logical Pointers(PKey)|
|HYRise|2010|MVOCC|Append-Only(N2O)|-|Physical Pointers|
|Hekaton|2011|MVOCC|Append-Only(O2N)|Tuple-Level(COOP)|Physical Pointers|
|MemSQL|2012|MVOCC|Append-Only(N2O)|Tuple-Level(VAC)|Physical Pointers|
|SAP HANA|2012|MV2PL|Time-Travel|Hybrid|Logical Pointers(Tupled)|
|NuoDB|2013|MV2PL|Append-Only(N2O)|Tuple-Level(VAC)|Logical Pointers(PKey)|
|HyPer|2015|MVOCC|Delta|Transaction-Level|Logical Pointers(Tupled)|
|||||||

但在DBMS中实现多版本控制有不同的方法，每种方法都会产生额外的计算和存储. 这些设计决策也高度依赖于彼此. 

在接下来的部分中，我们将讨论这些设计决策的实现问题和性能折衷. 之后在第7节对它们进行综合评估.

### 2-2 DBMS 元数据 [Meta-Data]



## 0x03 并发控制协议

### 3-1 MVTO 时戳排序 [Timestamp Ordering]

### 3-2 MVOCC 乐观并发控制 [Optimistic Concurrency Control]

### 3-3 MV2PL 两阶段锁 [Two-phase Locking]

### 3-4 序列化认证 [Serialization Certifier]

### 3-5 讨论



## 0x04 版本存储

### 4-1 Append-only Storage

### 4-2 Time-Travel Storage

### 4-3 Delta Storage

### 4-4 讨论



## 0x05 垃圾回收

### 5-1 Tuple-level Garbage Collection

### 5-2 Transaction-level Garbage Collection

### 5-3 讨论



## 0x06 索引管理

### 6-1 Logical Pointers

### 6-2  Physical Pointers

### 6-3 讨论