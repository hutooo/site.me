---
title: 停机问题
author: ash
tags: ["停机问题", "Computation", "计算机科学", "Computer Science"]
categories: ["计算探秘", "Explore"]
date: 2021-07-03T14:49:08+08:00
cover: "/images/hollow-knight.png"
---

> 我们都见过计算机屏幕上出现一个代表忙碌的小沙漏，不知道这是代表计算机死机了，还是在进行长时间的计算. 用户该马上重启机器呢，还是再等一会儿呢？ 如果能有一个算法，告诉我们计算机是不是陷入了某种无穷循环该有多好啊！那是很好，但那也是不可能的. --- <可能与不可能的边界>

# 0x00 Undecidable Decision Problem 不可解问题

## 什么是不可解问题?

不可解问题，并不是需要花大量时间才能求解的问题，也不是本来就无解的问题，更不是目前谁都不知道解法的未解决问题. 它指的是这样一种问题：

**它在原则上无法用程序来描述，也找不到一个正确的算法来解决.**

虽然不可思议，但这种问题被证明确实是存在的. 图灵在1936年提出了第一个不可解问题的实例: The Halting Problem / 停机问题.

## 存在不可解问题

考虑这样的一个函数集合，集合中的每一个函数，都要求输入一个1以上的整数，输出值也为整数. 这个函数集合是不可数的.

同时我们也知道，程序的集合是可数的, 因此在这样一个函数集合中，一定存在无法用程序表达的函数.

也即：函数的个数 比 程序的个数 要多.

# 0x01 停机问题

回到停机问题，它问的是，输入一段程序代码和一个针对此程序的输入，判断这个程序是否会在有限时间内终止.

## 程序的行为

1. 程序在有限时间内结束运行

```s
|数据 data| ---> |程序 program| ---> |输出结果，程序停止运行|
```

> 这里的有限时间，不管是1秒还是1亿年都无所谓，只要有终止之时就是有限时间内. 如果程序出错并结束运行，也算在有限时间内.

2. 程序永不结束运行

```s
|数据 data| ---> |程序 program| ---> |程序运行永不结束|
```

永不结束运行的程序栗子:

```c
// 无限循环程序
while (1 > 0) {

}
```
```c
while (x > 0) {

}

// 这段代码在 变量x 大于0 的情况下，才会无限循环~
// 因此判断程序是否结束运行，还需要对输入数据进行考量.
```

这里我们将判断程序命名为 **HaltCheck**

```s
------
|数据d| ----|
------     |      ----------
           |---> |          |     |判断将数据d  |
                 | HaltCheck|---> |输入程序p后  |
           |---> |          |     |是否会结束运行|
------     |      ----------
|程序p| ----|
------
```

简单想一下的话，似乎要写出 **HaltCheck** 也是非常困难的.

* 首先 **HaltCheck** 程序本身必须在有限时间内终止，否则就无法进行判断.
* 其次，**HaltCheck** 也无法通过实际运行程序p来判断，因为程序p可能永远不会停止.

实际上 **HaltCheck** 确实无法写出来.

## 证明

依然通过反证法来证明.

那么第一步，假设命题的否定形式成立，即：**可以写出HaltCheck程序.**

既然能写出HaltCheck，我们就可以将数据d和程序p输入其中. 因此一定能定义这样的函数:

```js
function HaltCheck(p, d) -> boolean

// HaltCheck(p, d)的结果为 true 或者 false
// 返回true， 则表示程序会在有限时间内结束
// 返回false，则表示程序会一直运行，永不停止
```

在 **HaltCheck** 的基础上，我们再编写一个程序：

```js
function RevHC(p) {
    halts = HaltCheck(p, p)
    if (halts) {
        while (1 > 0) {

        }
    }
}
```

**RevHC** 会用给定的程序p，判断 **HaltCheck(p, p)** 的结果. 如果为true, 那么 **RevHC** 会陷入无限循环.
如果为false, 那么 **RevHC** 就会终止.

可以发现 **RevHC** 和 **HaltCheck** 程序正好相反..

* 如果一个程序将自身作为数据运行，会在有限时间内终止，那么传入 **RevHC** 后，则会永远运行.
* 如果一个程序将自身作为数据运行，永不终止，那么传入 **RevHC** 后，则会在有限时间内终止.

现在将 **RevHC** 函数，传入 **RevHC** 中. 则有两种情况:

* 第一种 **RevHC(RevHC)** 会在有限时间内终止.

    - 这种情况，说明 **HaltCheck(RevHC, RevHC)** 返回的结果是false.
    - 但是 **HaltCheck(RevHC, RevHC)** 返回false 意味着 **RevHC(RevHC)** 永不终止.
    - 这里讨论的是 **RevHC(RevHC)** 会在有限时间内终止，但最后推导出的却是 **RevHC(RevHC)** 永不终止，产生了矛盾.

* 第二种 **RevHC(RevHC)** 永不终止.

    - 这种情况，说明 **HaltCheck(RevHC, RevHC)** 返回的结果是true.
    - 但是 **HaltCheck(RevHC, RevHC)** 返回true 意味着 **RevHC(RevHC)** 会在有限时间内终止.
    - 这里讨论的是 **RevHC(RevHC)** 永不终止，但最后推导出的却是 **RevHC(RevHC)** 会在有限时间内终止，产生了矛盾.

两种情况的讨论都是矛盾的~

因此证得，无法写出 **HaltCheck** 这样的程序.

## 一种感性的理解

假设我们能写出 **HaltCheck** 这样的程序，我们就能解决很多世界难题.

> 费马大定理: 当整数n > 2时，关于x,y,z的不定方程 $x^n+y^n=z^n$ 没有正整数解.

> 1994年，怀尔斯证明了费马大定理.

如果有了 **HaltCheck** ，我们就可以编写这样的程序：

```js
// 费马定理 伪代码

FermatCheck(k) {
    while (k > 0) {
        // 随意选择一些整数x,y,z,n, 其中 n > 2
        // 然后判断...
        if (x^n + y^n == z^n) {
            // 输出 x y z n
            // 结束程序
        }
    }
}
```

然后我们利用 **HaltCheck** 来判断:

```js
HaltCheck(FermatCheck, 1)

// 如果结果为true, 那么FermatCheck会在有限时间内终止.
// 表明费马大定理存在反例.

// 如果返回false, 那么FermatCheck永不终止.
// 验证了费马大定理.
```

当然，类似的 哥德巴赫猜想 也可以使用 **HaltCheck** 来进行验证.

> 哥德巴赫猜想: 任意一个大于3的偶数，都能写成两个质数之和.
