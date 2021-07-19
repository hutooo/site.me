---
title: B+树
author: ash
tags: ["DataStructure", "BPlusTree", "B+树"]
categories: ["数据结构"]
date: 2021-06-06T14:46:02+08:00
cover: "/images/page/touhou-lite.png"
---

## 起因

最近在研究数据库的一些实现，想着DIY一个~ 

虽然早就从大名鼎鼎的MySQL知识中了解到，经典的索引结构采用了B+树，但是真正涉及到实现细节，还是有点烦乱~

所以这篇文章，也就是一个B+树的实现和分析笔记了.

## 直接开始吧~

